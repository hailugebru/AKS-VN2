# VN2 Step-by-Step Guide for This Workspace

This guide creates the Azure network required for Virtual Nodes on Azure Container Instances (VN2), creates an AKS cluster that can use it, assigns the required managed identity permissions, installs the VN2 Helm chart from the upstream Microsoft repository, and validates pod scheduling. It also shows how to generate the confidential container policy annotation for VN2 workloads.

This guide is intentionally aligned to the following source material:

- Microsoft upstream VN2 repository: `microsoft/virtualnodesOnAzureContainerInstances`
- Upstream install section: `#installing-the-virtual-node-application-via-helm`
- Upstream pod customization guidance: `Docs/PodCustomizations.md#confidential-containers`
- Microsoft TechCommunity migration article for next-generation virtual nodes on ACI

This README is written for this workspace and its file layout. It does **not** claim to be the only supported deployment pattern. For current feature support, region availability, and limitations, Microsoft documentation remains the source of truth.

## What this guide assumes

- You are running PowerShell on Windows.
- Azure CLI, Helm, Git, and kubectl are already installed.
- You have permission to create resource groups, VNets, AKS clusters, and role assignments.
- You are using AKS with **Azure CNI** networking.
- The local workspace contains both folders side by side:
  - `AKS-VN2`
  - `virtualnodesOnAzureContainerInstances`

If you do not already have the upstream VN2 repo locally, clone it first:

```powershell
cd ..
git clone https://github.com/microsoft/virtualnodesOnAzureContainerInstances.git
cd .\AKS-VN2
```

## Files used by this guide

- `azure_cli.cli`: creates the resource group, VNet, `default` subnet, `aks` subnet, and delegated `cg` subnet.
- `hello-world-pod.yaml`: sample pod that targets VN2.
- `..\virtualnodesOnAzureContainerInstances\Helm\virtualnode`: local Helm chart path from the upstream Microsoft repo.

## Source-aligned notes before you start

- VN2 requires **Azure CNI**.
- The delegated subnet for VN2-backed ACI container groups must be delegated to `Microsoft.ContainerInstance/containerGroups`.
- The subnet name `cg` is used by the upstream chart defaults. If you change it, you may also need to adjust Helm values.
- The upstream VN2 repo recommends AKS worker nodes with at least **4 vCPU and 16 GiB RAM** so the VN2 infrastructure pods have enough capacity.
- Kubernetes version support changes over time. Do **not** assume the example version below is available in every region; verify with `az aks get-versions` before cluster creation.
- Confidential containers for VN2 are region-dependent. Validate support in your chosen region before relying on that path.

## Step 1: Sign in and set variables

Open PowerShell in the `AKS-VN2` folder and define the variables used by the commands below.

```powershell
az login

$SubscriptionId = "<your-subscription-id>"
$ResourceGroup = "rg-aks-vnet"
$Location = "eastus2"
$VnetName = "vnet-aks"
$AksSubnetName = "aks"
$CgSubnetName = "cg"
$AksClusterName = "aks-vn2-demo"
$KubernetesVersion = "<supported-1.33.x-version-in-your-region>"
$NodeVmSize = "Standard_D4s_v5"
$NodeCount = 2
$SystemNodeOsDiskSize = 128
$ReleaseName = "virtualnode"

az account set --subscription $SubscriptionId
az provider register --namespace Microsoft.ContainerInstance
```

Notes:

- The `cg` subnet name matters because it matches the upstream chart default behavior.
- The `cg` subnet must be delegated to `Microsoft.ContainerInstance/containerGroups`.
- The `default` subnet is reserved as `10.0.0.0/24` in the upstream guidance.

## Step 2: Create the VNet and subnets

Run the existing script in this folder.

```powershell
.\azure_cli.cli
```

That script creates:

- VNet address space: `10.0.0.0/8`
- `default` subnet: `10.0.0.0/24`
- `aks` subnet: `10.1.0.0/16`
- `cg` subnet: `10.2.0.0/16`
- Delegation on `cg`: `Microsoft.ContainerInstance/containerGroups`

If you want the raw Azure CLI commands instead of running the script, this is the equivalent:

```powershell
az group create --name $ResourceGroup --location $Location

az network vnet create `
  --resource-group $ResourceGroup `
  --name $VnetName `
  --address-prefixes 10.0.0.0/8 `
  --subnet-name default `
  --subnet-prefixes 10.0.0.0/24

az network vnet subnet create `
  --resource-group $ResourceGroup `
  --vnet-name $VnetName `
  --name $AksSubnetName `
  --address-prefixes 10.1.0.0/16

$CgSubnetId = az network vnet subnet create `
  --resource-group $ResourceGroup `
  --vnet-name $VnetName `
  --name $CgSubnetName `
  --address-prefixes 10.2.0.0/16 `
  --delegations Microsoft.ContainerInstance/containerGroups `
  --query id -o tsv
```

Validate the subnet layout:

```powershell
az network vnet subnet list `
  --resource-group $ResourceGroup `
  --vnet-name $VnetName `
  --query "[].{Name:name,Prefix:addressPrefix,Delegation:delegations[0].serviceName}" `
  --output table
```

Expected result: `cg` appears with delegation to `Microsoft.ContainerInstance/containerGroups`.

## Step 3: Get the AKS subnet ID

```powershell
$AksSubnetId = az network vnet subnet show `
  --resource-group $ResourceGroup `
  --vnet-name $VnetName `
  --name $AksSubnetName `
  --query id -o tsv

if (-not $CgSubnetId) {
  $CgSubnetId = az network vnet subnet show `
    --resource-group $ResourceGroup `
    --vnet-name $VnetName `
    --name $CgSubnetName `
    --query id -o tsv
}
```

## Step 4: Verify supported AKS versions and create the cluster

List supported versions first:

```powershell
az aks get-versions --location $Location --output table
```

Then create the cluster.

VN2 requires AKS with Azure CNI networking and enough node capacity to host the VN2 infrastructure pods. The upstream repo recommends at least 4 vCPU and 16 GiB worker nodes.

```powershell
az aks create `
  --resource-group $ResourceGroup `
  --name $AksClusterName `
  --location $Location `
  --enable-managed-identity `
  --network-plugin azure `
  --vnet-subnet-id $AksSubnetId `
  --service-cidr 10.3.0.0/24 `
  --dns-service-ip 10.3.0.10 `
  --node-vm-size $NodeVmSize `
  --node-count $NodeCount `
  --node-osdisk-size $SystemNodeOsDiskSize `
  --kubernetes-version $KubernetesVersion `
  --generate-ssh-keys
```

## Step 5: Grant the required Azure permissions

This is the step most likely to break the deployment if skipped or scoped incorrectly.

The upstream VN2 documentation requires the AKS managed identity used by VN2 to have permission to manage the ACI-backed resources in the AKS-managed resource group. If the VNet is outside that managed resource group, network permissions are also required for the VNet side.

This guide uses the cluster kubelet identity as exposed by `identityProfile.kubeletidentity`, and scopes the network permission to the delegated VN2 subnet.

```powershell
$NodeResourceGroup = az aks show `
  --resource-group $ResourceGroup `
  --name $AksClusterName `
  --query nodeResourceGroup -o tsv

$NodeResourceGroupId = az group show `
  --name $NodeResourceGroup `
  --query id -o tsv

$KubeletIdentityResourceId = az aks show `
  --resource-group $ResourceGroup `
  --name $AksClusterName `
  --query "identityProfile.kubeletidentity.resourceId" -o tsv

$KubeletIdentityPrincipalId = az identity show `
  --ids $KubeletIdentityResourceId `
  --query principalId -o tsv

az role assignment create `
  --assignee-object-id $KubeletIdentityPrincipalId `
  --assignee-principal-type ServicePrincipal `
  --role Contributor `
  --scope $NodeResourceGroupId

az role assignment create `
  --assignee-object-id $KubeletIdentityPrincipalId `
  --assignee-principal-type ServicePrincipal `
  --role "Network Contributor" `
  --scope $CgSubnetId
```

Why these permissions are needed in this guide:

- `Contributor` on the AKS-managed resource group allows VN2 to create and manage the ACI-backed container group resources it needs.
- `Network Contributor` on the delegated `cg` subnet allows VN2-backed ACI container groups to join that subnet.

If role assignment propagation is slow, wait a minute or two and retry the Helm install.

## Step 6: Optional least-privilege alternative to broad Contributor

If you do not want to grant broad `Contributor` access, the upstream security guidance describes a smaller custom role for the core VN2 operations.

Create `role.json`:

```powershell
$TargetRgName = $NodeResourceGroup
$VnetRgName = $ResourceGroup

@"
{
  "Name": "virtualnodeCustomRole",
  "Description": "Minimum permissions required for VN2 core operations",
  "Actions": [
    "Microsoft.ContainerInstance/containerGroups/read",
    "Microsoft.ContainerInstance/containerGroups/write",
    "Microsoft.ContainerInstance/containerGroups/delete",
    "Microsoft.ContainerInstance/containerGroups/containers/exec/action",
    "Microsoft.Network/virtualNetworks/subnets/join/action"
  ],
  "AssignableScopes": [
    "/subscriptions/$SubscriptionId/resourceGroups/$TargetRgName",
    "/subscriptions/$SubscriptionId/resourceGroups/$VnetRgName"
  ]
}
"@ | Set-Content -Path .\role.json

az role definition create --role-definition "@role.json"

az role assignment create `
  --assignee-object-id $KubeletIdentityPrincipalId `
  --assignee-principal-type ServicePrincipal `
  --role virtualnodeCustomRole `
  --scope "/subscriptions/$SubscriptionId/resourceGroups/$TargetRgName"

az role assignment create `
  --assignee-object-id $KubeletIdentityPrincipalId `
  --assignee-principal-type ServicePrincipal `
  --role virtualnodeCustomRole `
  --scope "/subscriptions/$SubscriptionId/resourceGroups/$VnetRgName"
```

Use either the built-in role approach from Step 5 or this custom role approach, not both unless you are testing.

## Step 7: Get kubeconfig and confirm cluster access

```powershell
az aks get-credentials --name $AksClusterName --resource-group $ResourceGroup --overwrite-existing

kubectl get nodes
```

At this point you should only see the regular AKS VM-backed nodes.

## Step 8: Install the VN2 Helm chart from the upstream repo

```powershell
helm install $ReleaseName ..\virtualnodesOnAzureContainerInstances\Helm\virtualnode
```

Wait for the VN2 components to become ready:

```powershell
kubectl get pods -n vn2
kubectl get nodes
```

Expected result: a new node with a name similar to `virtualnode-0` shows as `Ready`.

## Step 9: Deploy a test pod to VN2

This workspace already contains a sample pod manifest with the VN2 selector and toleration.

```powershell
kubectl apply -f .\hello-world-pod.yaml
kubectl get pod demo-pod -o wide
kubectl logs demo-pod
```

The pod should land on the VN2 node, not on a regular AKS VM node.

## Step 10: Required scheduling settings for your own workloads

For workloads that must run on VN2, use:

```yaml
nodeSelector:
  virtualization: virtualnode2
  kubernetes.io/os: linux
tolerations:
- effect: NoSchedule
  key: virtual-kubelet.io/provider
  operator: Exists
```

This is the updated VN2 scheduling model. It is different from the legacy AKS virtual-node add-on.

## Step 11: Optional confidential containers on VN2

A VN2 pod becomes confidential when you add the `microsoft.containerinstance.virtualnode.ccepolicy` annotation containing the generated CCE policy.

Do not hand-author that policy. Generate it with `confcom`.

```powershell
az extension add -n confcom
az confcom acipolicygen --virtual-node-yaml .\hello-world-pod.yaml
```

That command pulls the referenced images, hashes their layers, generates the CCE policy, and injects the annotation into the YAML.

Then apply the updated manifest:

```powershell
kubectl apply -f .\hello-world-pod.yaml
```

Important notes:

- Confidential VN2 support is region-dependent.
- The annotation value is generated policy, not a manual toggle string.
- The upstream pod customization document is the source of truth for supported confidential-container settings and other VN2 pod customizations.

## Optional: Private ACR image pull with Managed Identity

The migration guidance calls out support for image pulling through Private Link and Managed Identity. The upstream VN2 chart supports this through `acrTrustedAccess` values.

If you are using a private ACR with Trusted Access enabled:

1. Create or reuse a user-assigned managed identity.
2. Grant that identity `AcrPull` on the target ACR.
3. Set these Helm values:

```yaml
acrTrustedAccess:
  enabled: true
  identityResourceId: /subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<miName>
  identityPrincipalId: <managed-identity-principal-id>
```

To apply those values without editing the chart defaults:

```powershell
helm upgrade --install $ReleaseName ..\virtualnodesOnAzureContainerInstances\Helm\virtualnode `
  --set acrTrustedAccess.enabled=true `
  --set acrTrustedAccess.identityResourceId="/subscriptions/<subId>/resourceGroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<miName>" `
  --set acrTrustedAccess.identityPrincipalId="<managed-identity-principal-id>"
```

## Troubleshooting checks

If VN2 does not come up cleanly, run these first:

```powershell
kubectl get pods -n vn2
kubectl logs -n vn2 virtualnode-0 -c proxycri
az role assignment list --assignee $KubeletIdentityPrincipalId --all --output table
az network vnet subnet show --resource-group $ResourceGroup --vnet-name $VnetName --name $CgSubnetName --query "{Name:name,Delegations:delegations}" --output json
```

Common causes:

- The `cg` subnet was not delegated.
- The required role assignments have not propagated yet.
- The AKS nodes are too small to host the VN2 infrastructure pods.
- The cluster was created with a networking mode other than Azure CNI.
- The Kubernetes version selected is not actually supported in the chosen region.

## Cleanup

To remove the demo pod:

```powershell
kubectl delete -f .\hello-world-pod.yaml
```

To delete the whole environment:

```powershell
az group delete --name $ResourceGroup --yes --no-wait
```

## Summary

If you follow the steps above in order, you will end with:

- A VNet created by `azure_cli.cli`
- An AKS cluster attached to the `aks` subnet using Azure CNI
- A delegated `cg` subnet for VN2-backed ACI pods
- The Azure permissions required for VN2 in this workspace pattern
- A Helm-installed VN2 node in `Ready` state
- A sample pod running on VN2
- An optional path to generate and apply a confidential-container policy for VN2 workloads
