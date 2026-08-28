# Virtual nodes on Azure Container Instances: a new compute layer for AKS

*Effortless burst capacity, confidential containers, and a serverless compute layer for AKS, without leaving the Kubernetes you already know.*

---

## Meet virtual nodes on ACI

Azure Kubernetes Service (AKS) gives you managed Kubernetes: the full Kubernetes API without operating the control plane yourself. **Virtual nodes on Azure Container Instances** go a step further, letting your pods run directly on Azure's serverless container platform, with the elasticity and per-second economics of ACI and with no capacity planning and no waiting for machines. Whether you already run AKS or want a managed Kubernetes that bursts without node management, this is for you.

In short: **virtual nodes on ACI attach Azure's serverless container platform to your cluster as Kubernetes nodes.** PPods run as Hyper-V isolated containers, sized per pod rather than packed onto a fixed VM, up to 200 pods per virtual node. Run multiple virtual nodes, scaled as replicas, for more. They behave like any other pod: same `kubectl`, Helm, and GitOps.

Kubernetes has always assumed a fixed set of machines underneath it. That assumption shapes everything above it: you size a node pool for a specific VM type in a specific region, you plan for peak rather than for average, and every workload on a node shares the same kernel and the same security boundary. Virtual nodes on ACI relax that assumption, which is what makes both elastic capacity and per container isolation possible without a different Kubernetes.

If you've used the original AKS virtual nodes add-on (Virtual Kubelet based), this is **not a rebrand.** It is a new implementation that integrates far more deeply with Kubernetes, lifts most prior limitations (init containers, persistent volumes, managed identity, richer networking), and adds confidential containers as a first-class capability. The [migration guide on the Apps on Azure blog](https://techcommunity.microsoft.com/blog/appsonazureblog/migrating-to-the-next-generation-of-virtual-nodes-on-azure-container-instances-a/4496565) has the details.

Two capabilities carry the rest of this post: effortless burst capacity, and confidential containers.

---

## How virtual nodes on ACI work

ACI runs every container as a Hyper-V isolated container, which means each one gets its own lightweight virtual machine boundary rather than sharing a kernel with its neighbors. Azure operates that platform. A virtual node connects it to your cluster.

The cluster's control plane, the component that decides where each container runs, sees two kinds of destination: a small pool of virtual machines carrying cluster services, and one or more virtual nodes. 

<img width="1700" height="800" alt="Diagram showing the AKS control plane scheduling to a system node pool and to virtual nodes, which hand pods off to the ACI serverless platform." src="https://github.com/user-attachments/assets/0d2bab66-e9ca-42b8-8a6b-786f3d59f84d" />
  
  > *`Diagram showing the AKS control plane scheduling to a system node pool and to virtual nodes, which hand pods off to the ACI serverless platform.*

From the application manifest's perspective, nothing changes. The pod lands on a virtual node; the virtual node hands it off to ACI. See [Microsoft Learn: virtual nodes on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) for the official capability and current limits.

---

## Virtual nodes on ACI in practice

The rest of this post is hands on. You do not need to be a Kubernetes expert to follow it. `kubectl` is the command line tool for talking to a cluster, Helm installs packaged software into one, and a manifest is a text file describing what you want to run. If you have a cluster, everything below runs against it as written.

The manifests behind the examples live in a companion demo repo. Setup is documented officially, and you can reproduce this end to end from the ACI virtual nodes documentation and the `microsoft/virtualnodesOnAzureContainerInstances` Helm repo.

One requirement before you start: deploy into a delegated ACI subnet, meaning a subnet in your virtual network set aside for the ACI platform to place containers in. Size it for peak pod count plus headroom, since every pod consumes an address from it for its lifetime.

Demo manifests: https://github.com/hailugebru/AKS-VN2/tree/main(opens in new window) (a personal sample repo, provided as is and not a supported Microsoft artifact).

### Enable virtual nodes on ACI

The virtual node is deployed via Helm. The Microsoft GitHub repo is itself a Helm repository, so a single `helm install` is all you strictly need. Cloning first, shown here, just makes it easier to customize values. Running `kubectl get nodes` afterward confirms the node registered.

```bash
git clone https://github.com/microsoft/virtualnodesOnAzureContainerInstances.git
helm install <yourReleaseName> ./virtualnodesOnAzureContainerInstances/Helm/virtualnode
kubectl get nodes
```

The virtual node appears alongside any existing capacity, ready to accept work.

<img width="485" height="77" alt="kubectl get nodes showing the virtual node registered alongside the system node pool." src="https://github.com/user-attachments/assets/a0c141f3-4b44-4f87-bdcb-56b51e52288d" />

> *Image 1: `kubectl get nodes` showing the virtual node registered alongside the system node pool.*

### A virtual node is a Kubernetes node

You target it the same way you would target any node. These few lines in a manifest say "run this on the virtual node":

```yaml
nodeSelector:
  virtualization: virtualnode2
  kubernetes.io/os: linux
tolerations:
- key: virtual-kubelet.io/provider
  operator: Exists
  effect: NoSchedule
```

That is the entire integration surface. No new API to learn, no separate deployment pipeline, no application changes. `kubectl describe`, `kubectl logs`, and `kubectl exec`, the standard commands for inspecting and troubleshooting, all work as they would anywhere else, including opening a shell inside a container running in a Hyper-V isolated boundary.

Logs and metrics flow through the same path you already use, so existing dashboards and alerts keep working. Confirm the specifics for your monitoring stack against the documentation, since the pods are not running on a VM you own.

<img width="1402" height="490" alt="kubectl get pods -o wide after scaling, every replica on the virtual node, no additional VMs." src="https://github.com/user-attachments/assets/f7842539-7736-4d4c-89d0-9b87bff6e014" />
> *Image 2: `kubectl get` / `kubectl logs` / `kubectl exec` against a virtual-node-hosted pod.*

Scaling stays trivial. `kubectl scale deployment demo-deployment --replicas=10` lands every replica on the same virtual node, with no VMSS scale event, no provisioning latency, no climbing node-count chart. The same flow scales just as cleanly to hundreds.

<img width="1402" height="490" alt="image" src="https://github.com/user-attachments/assets/f7842539-7736-4d4c-89d0-9b87bff6e014" />

> *Image 3: `kubectl get pods -o wide` after scaling, every replica on the virtual node, no additional VMs.*

### One annotation makes a pod confidential

Turning a regular container into a confidential one takes a single addition to its manifest: a policy that pins exactly which images, commands, environment variables, mounts, and capabilities are permitted inside the Trusted Execution Environment. The format is a base64 encoded Rego document, called a CCE (Container Confidential Enforcement) policy.

You do not write that policy by hand. A tool generates it from the manifest you already have:
```bash
az extension add -n confcom
az confcom acipolicygen --virtual-node-yaml ./hello-world-deployment.yaml
```

The tool pulls each image, hashes its layers, builds the allow-list, and injects the annotation back into the manifest. `kubectl apply`, and you're done. (`acipolicygen` has prerequisites of its own, including a working Docker installation; see the confcom documentation.)

<img width="1578" height="418" alt="az confcom acipolicygen pulling and hashing images, emitting the base64 policy." src="https://github.com/user-attachments/assets/fe844450-6423-4ad3-baff-b0f4c7c13925" />

> *Image 4: `az confcom acipolicygen` pulling and hashing images, emitting the base64 policy.*

Here is why this is a genuinely new isolation primitive rather than a stronger version of an existing one. Most container security policy is enforced by software in the cluster, which means an attacker who compromises the host can potentially bypass it. This policy is enforced by the guest operating system inside the TEE instead. The underlying hardware, AMD SEV-SNP, also produces an attestation report, retrievable from inside the container, which is a cryptographic proof that the workload running is the workload you specified and nothing tampered with it. That is the guarantee regulated industries have been asking for, and increasingly the one AI workloads running untrusted code need too. 

Background: [Microsoft Learn: confidential containers on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-confidential-overview).

---

## Wrapping up

Virtual nodes on ACI give containers on Azure two things that were previously hard to deliver cleanly on Kubernetes:

* **Effortless burst capacity** on Azure's serverless container platform, billed per second for the cores and memory used, with no capacity planning and no waiting for machines.
* **Confidential containers** with hardware attested, per container isolation inside a Trusted Execution Environment.

Virtual nodes are additive, not a replacement. Traditional node pools remain the right home for steady state, DaemonSet, and persistent volume workloads, and AKS features such as Node Auto Provisioning and Virtual Machine Node Pools already make that baseline more flexible. Virtual nodes on ACI absorb the spikes, the short-lived jobs, and the specialized isolation work on top.

### Where to start

* **New to containers on Azure?** Start with a small AKS cluster and add a virtual node from day one. You get a managed Kubernetes environment without having to guess your peak capacity in advance, and the elastic layer is there the first time you need it.
* **Already running AKS?** Add a virtual node to an existing cluster and move one bursty or short lived workload to it. Nothing else changes, and the comparison is immediate.
* **Evaluating platforms?** The capability that is hard to find elsewhere is the confidential containers path: hardware attested isolation per container, reachable through a standard Kubernetes manifest.

The result: **virtual nodes on ACI expand what AKS can run, with more capacity, stronger isolation, and per-second economics, without changing the Kubernetes operating model you already use.** Same `kubectl`, same manifests, same GitOps. New ceiling.

For the high-level overview, official documentation, and Helm details, the [migration guide on the Apps on Azure blog](https://techcommunity.microsoft.com/blog/appsonazureblog/migrating-to-the-next-generation-of-virtual-nodes-on-azure-container-instances-a/4496565) and [Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) are the sources of truth. The companion repo holds the demo manifests used in this post.
