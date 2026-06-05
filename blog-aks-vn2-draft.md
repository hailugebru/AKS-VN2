# Virtual nodes on Azure Container Instances: A Powerful New Way to Run AKS

*Effortless burst capacity, confidential containers, and a serverless compute layer for AKS, without leaving the Kubernetes you already know.*

---

## Meet virtual nodes on ACI

Azure Kubernetes Service (AKS) gives you managed Kubernetes: the full Kubernetes API without operating the control plane yourself. **Virtual nodes on Azure Container Instances** go a step further, letting your pods run directly on Azure's serverless container platform, with the elasticity and per-second economics of ACI and no VM node pools to size or manage. Whether you already run AKS or want a managed Kubernetes that bursts without node management, this is for you.

In short: **virtual nodes on ACI attach Azure's serverless container platform to your cluster as Kubernetes nodes.** Pods run as Hyper-V isolated containers with effectively unlimited cores and memory, up to 200 pods per node today (`--max-pods`, being raised); run multiple virtual nodes, scaled as replicas, for more. They behave like any other pod: same `kubectl`, Helm, and GitOps.

If you've used the original AKS virtual nodes add-on (Virtual Kubelet based), this is **not a rebrand.** It is a new implementation that integrates far more deeply with Kubernetes, lifts most prior limitations (init containers, persistent volumes, managed identity, richer networking), and adds confidential containers as a first-class capability. The [migration guide on the Apps on Azure blog](https://techcommunity.microsoft.com/blog/appsonazureblog/migrating-to-the-next-generation-of-virtual-nodes-on-azure-container-instances-a/4496565) has the details.

Two things stand out:

1. **Effortless burst capacity.** Schedule up to 200 pods per virtual node in seconds (limit being raised), and run multiple virtual nodes for more, with no node pool sizing, no autoscaler wait, and no idle-VM cost.
2. **Confidential containers.** Hardware-attested, per-container isolation inside a Trusted Execution Environment (TEE) that opens up regulated, sovereign, and AI-on-untrusted-code workloads on AKS.

---

## What customers have been asking for

Most AKS capacity conversations come back to one thing: **a node pool is tied to a specific VM SKU, region, and zone.** When demand spikes, or capacity in that exact SKU/region/zone is momentarily tight, you hit familiar allocation errors (`SkuNotAvailable`, `AllocationFailed`, `ZonalAllocationFailed`, quota exceeded). The choice becomes overprovision and carry idle VMs, or stay lean and risk not scaling when you need to.

AKS already widens that surface for steady-state workloads: **Node Auto Provisioning** picks the right SKU dynamically, and **Virtual Machine Node Pools** span multiple SKUs ([details on the AKS Engineering Blog](https://blog.aks.azure.com/2025/12/06/node-auto-provisioning-capacity-management)). **Virtual nodes on ACI are complementary.** For bursty, event-driven, short-lived, or confidential workloads, the pod lands on Azure's serverless platform directly, sidestepping VM allocation entirely. NAP and VM Node Pools handle the baseline; virtual nodes absorb the spikes and the specialized isolation work on top.

---

## How virtual nodes on ACI work

ACI runs every container as a Hyper-V isolated container on a massive shared platform that Azure operates. Plug that into AKS through a virtual node and the picture looks like this:

```
                ┌──────────────────────────────────────────┐
                │           AKS Control Plane              │
                │  (kubectl, Helm, Argo CD, Flux, KEDA…)   │
                └──────────────────────────────────────────┘
                                  │
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
   ┌─────────────────────┐               ┌────────────────────────┐
   │ System node pool    │               │  Virtual node on ACI   │
   │ (small VM, control  │               │  (one or more,         │
   │  plane add-ons)     │               │   scaled as replicas)  │
   └─────────────────────┘               └─────────┬──────────────┘
                                                   │
                                                   ▼
                                   ┌──────────────────────────────┐
                                   │   ACI serverless platform    │
                                   │   (Hyper-V isolated          │
                                   │    container per pod)        │
                                   └──────────────────────────────┘
```

From the manifest's perspective, nothing changes. The pod lands on a virtual node; the virtual node hands it off to ACI. See [Microsoft Learn: virtual nodes on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) for the official capability and current limits.

---

## What virtual nodes on ACI unlock

| | |
|---|---|
| **Effortless burst capacity** | Up to 200 pods per virtual node in seconds (`--max-pods`, being raised), and multiple virtual nodes for more. No VMSS cold-start tax, no node pool sizing exercise. |
| **Per-second billing for cores and memory** | Batch jobs, CI agents, KEDA-scaled handlers, and AI inference pay only for the vCPU and memory they consume, by the second. |
| **Confidential containers** | Each pod runs inside a hardware-based, attested Trusted Execution Environment (TEE) that protects data in use and encrypts memory, with verifiable execution policies and a hardware root of trust through guest attestation. Confidential containers on AKS sunset in March 2026; virtual nodes on ACI are the forward path. |
| **Multi-tenant isolation** | Per-pod hardware isolation makes multi-tenancy possible on a single AKS cluster, which standard AKS nodes cannot offer. Safe multi-tenancy and hostile-workload isolation still require additional boundaries, network protections in particular; Microsoft is publishing holistic guidance for configuring them. |

---

## Virtual nodes on ACI in practice

The exact manifests behind the screenshots below live in a companion demo repo. Setup itself is documented officially: you can reproduce this end to end from the [ACI virtual nodes documentation](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) and the [microsoft/virtualnodesOnAzureContainerInstances Helm repo](https://github.com/microsoft/virtualnodesOnAzureContainerInstances). Two requirements worth calling out before you start: deploy into a delegated ACI subnet, and keep AKS auto-upgrades to patch-only within a minor version, since minor-version jumps can ship breaking changes that regress the virtual nodes until the chart is realigned.

> **Demo manifests:** [https://github.com/hailugebru/AKS-VN2/tree/main](https://github.com/hailugebru/AKS-VN2/tree/main)

### Enable virtual nodes on ACI

The virtual node is deployed via Helm. The Microsoft GitHub repo is itself a Helm repository, so a single `helm install` is all you strictly need; cloning first (shown here) just makes it easier to customize values. `kubectl get nodes` afterward is an optional check that the node registered:

```bash
git clone https://github.com/microsoft/virtualnodesOnAzureContainerInstances.git
helm install <yourReleaseName> ./virtualnodesOnAzureContainerInstances/Helm/virtualnode
kubectl get nodes
```

The virtual node shows up alongside your existing node pools, ready to schedule pods.

<img width="485" height="77" alt="image" src="https://github.com/user-attachments/assets/a0c141f3-4b44-4f87-bdcb-56b51e52288d" />

> *[Screenshot: `kubectl get nodes` showing the virtual node registered alongside the system node pool.]*

### A virtual node is a Kubernetes node

You target it the same way you'd target any node:

```yaml
nodeSelector:
  virtualization: virtualnode2
  kubernetes.io/os: linux
tolerations:
- key: virtual-kubelet.io/provider
  operator: Exists
  effect: NoSchedule
```

`kubectl describe`, `kubectl logs`, and `kubectl exec` work as they would on any node, including a shell into a pod running as a Hyper-V isolated container.

<img width="960" height="322" alt="image" src="https://github.com/user-attachments/assets/3f5a6bf7-49d4-4850-927b-fc107f6c3c34" />

> *[Screenshot 1: `kubectl get` / `kubectl logs` / `kubectl exec` against a virtual-node-hosted pod.]*

Scaling stays trivial. `kubectl scale deployment demo-deployment --replicas=10` lands every replica on the same virtual node, with no VMSS scale event, no provisioning latency, no climbing node-count chart. The same flow scales just as cleanly to hundreds.

<img width="1402" height="490" alt="image" src="https://github.com/user-attachments/assets/f7842539-7736-4d4c-89d0-9b87bff6e014" />

> *[Screenshot 2: `kubectl get pods -o wide` after scaling, every replica on the virtual node, no additional VMs.]*

### One annotation makes a pod confidential

The single switch that turns a regular virtual-node pod into a confidential one is a pod annotation holding the CCE (Container Confidential Enforcement) policy, a base64 Rego document that pins exactly which images, commands, environment variables, mounts, and capabilities are permitted inside the TEE.

You don't write that policy by hand:

```bash
az extension add -n confcom
az confcom acipolicygen --virtual-node-yaml ./hello-world-deployment.yaml
```

The tool pulls each image, hashes its layers, builds the allow-list, and injects the annotation back into the manifest. `kubectl apply`, and you're done. (`acipolicygen` has prerequisites of its own, including a working Docker installation; see the confcom documentation.)

<img width="1578" height="418" alt="image" src="https://github.com/user-attachments/assets/fe844450-6423-4ad3-baff-b0f4c7c13925" />

> *[Screenshot 3: `az confcom acipolicygen` pulling and hashing images, emitting the base64 policy.]*

The detail that makes this a genuinely new isolation primitive: **the policy is enforced by the guest OS inside the TEE**, not by a control-plane admission webhook that an attacker on the host could bypass. The attestation report is retrievable from inside the pod, giving you a cryptographic *"prove to me you did not tamper with my workload"* guarantee that regulated industries, and increasingly AI workloads running untrusted code, have been asking for. Background: [Microsoft Learn: confidential containers on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-confidential-overview).

---

## Wrapping up

Virtual nodes on ACI give AKS two things that were previously hard to deliver cleanly on Kubernetes:

- **Effortless burst capacity** on Azure's serverless container platform, billed per second for the cores and memory used, with no node pool sizing and no autoscaler wait.
- **Confidential containers** with hardware-attested, per-container isolation inside a Trusted Execution Environment (TEE).

It's additive, not a replacement. Traditional node pools, NAP, and Virtual Machine Node Pools remain the right home for steady-state, DaemonSet, and persistent-volume workloads. Virtual nodes on ACI absorb the spikes, the short-lived jobs, and the specialized isolation work on top.

The result: **virtual nodes on ACI expand what AKS can run, with more capacity, stronger isolation, and per-second economics, without changing the Kubernetes operating model you already use.** Same `kubectl`, same manifests, same GitOps. New ceiling.

For the high-level overview, official documentation, and Helm details, the [migration guide on the Apps on Azure blog](https://techcommunity.microsoft.com/blog/appsonazureblog/migrating-to-the-next-generation-of-virtual-nodes-on-azure-container-instances-a/4496565) and [Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) are the sources of truth. The companion repo holds the demo manifests used in this post.
