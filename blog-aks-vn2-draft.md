# Virtual Nodes on Azure Container Instances (VN2) — A Powerful New Way to Run AKS

*Effortless burst capacity, Confidential Containers, and a serverless compute layer for AKS — without leaving the Kubernetes you already know.*

---

## Meet VN2

Azure Kubernetes Service is the platform our enterprise customers trust to run almost anything — line-of-business APIs, data pipelines, large-scale AI inference. **Virtual Nodes on Azure Container Instances (VN2)** is a powerful new capability inside AKS that meaningfully expands what those clusters can do.

In one sentence: **VN2 attaches Azure's shared, serverless container fleet to your AKS cluster as a single, large-scale Kubernetes node** (subject to ACI service limits and regional availability — see [Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/virtual-nodes) for the current matrix). Pods scheduled to it run as micro-VMs on Azure Container Instances. They appear and behave exactly like any other Kubernetes pod — same `kubectl`, same Helm charts, same Argo CD or Flux pipelines.

If you've used the original AKS virtual nodes (the Virtual Kubelet–based experience), it's worth being explicit: **VN2 is not a rebrand.** It's a new implementation that integrates much more deeply with Kubernetes, lifts most of the prior limitations (init containers, persistent volume support, managed identity, richer networking, and more), and adds **Confidential Containers** as a first-class capability. The original virtual nodes solved one problem — fast burst. VN2 makes virtual nodes a production-grade compute layer for AKS.

Two things VN2 brings to AKS that are worth getting excited about:

1. **Effortless burst capacity** — schedule hundreds of pods on a single virtual node in seconds, with no node-pool sizing, no autoscaler wait, and no idle-VM cost.
2. **Confidential Containers** — hardware-attested, per-container isolation that opens up regulated, sovereign, and AI-on-untrusted-code workloads on AKS.

The rest of this post walks through what VN2 is, what it unlocks, and what it looks like running.

---

## What customers have been asking for

The recurring conversations I have with enterprise customers about AKS keep landing on a few asks: stop guessing the VM SKU, skip the autoscaler wait, keep utilization high without manual tuning, run multi-tenant or sovereign workloads safely on a single cluster.

A lot of that nuance comes back to one thing: **a traditional node pool is tied to a specific VM SKU, region, and zone.** When demand spikes — or when Azure capacity in that exact SKU/region/zone is momentarily tight — that bond surfaces as familiar allocation errors (`SkuNotAvailable`, `AllocationFailed`, `ZonalAllocationFailed`, quota exceeded). The choice becomes overprovision and carry idle VMs, or stay lean and risk being unable to scale precisely when you need to. The [AKS Engineering Blog has a thorough walkthrough](https://blog.aks.azure.com/2025/12/06/node-auto-provisioning-capacity-management).

AKS already has strong answers here. **Node Auto Provisioning (NAP)**, built on Karpenter, picks the right VM SKU dynamically. **Virtual Machine Node Pools** let a single pool span multiple SKUs. Both meaningfully widen the allocation surface for steady-state workloads, and both remain the right starting point.

**VN2 is complementary.** For bursty, event-driven, short-lived, or confidential workloads, VN2 sidesteps the VM-allocation question entirely — the pod lands on Azure's shared ACI fleet directly. NAP and VM Node Pools handle the baseline; VN2 absorbs the spikes and the specialized isolation work on top.

---

## How VN2 works

ACI runs every container in its own micro-VM on a massive shared fleet that Azure operates. Plug that fleet into AKS through a virtual node and the picture looks like this:

```
                ┌──────────────────────────────────────────┐
                │           AKS Control Plane              │
                │  (kubectl, Helm, Argo CD, Flux, KEDA…)   │
                └──────────────────────────────────────────┘
                                  │
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
   ┌─────────────────────┐               ┌────────────────────────┐
   │ System node pool    │               │  VN2 virtual node      │
   │ (small VM, control- │               │  (single logical node, │
   │  plane add-ons)     │               │   backed by ACI fleet) │
   └─────────────────────┘               └─────────┬──────────────┘
                                                   │
                                                   ▼
                                   ┌──────────────────────────────┐
                                   │   ACI serverless fleet       │
                                   │   (micro-VM isolation        │
                                   │    per pod)                  │
                                   └──────────────────────────────┘
```

From the manifest's perspective, nothing changes. The pod lands on a virtual node; the virtual node hands it off to ACI. See [Microsoft Learn: Virtual Nodes on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) for the official capability and current limits.

---

## What VN2 unlocks

| | |
|---|---|
| **Effortless burst capacity** | Hundreds of pods on a single virtual node in seconds. No `--max-pods` ceiling, no VMSS cold-start tax, no node-pool sizing exercise. |
| **Per-second, per-pod billing** | Batch jobs, CI agents, KEDA-scaled handlers, and AI inference pay only for the pod-seconds they consumed. |
| **Confidential Containers** | Each pod runs in a hardware-encrypted micro-VM with cryptographic attestation — per-container, not per-VM. The [standalone Confidential Containers (preview) on AKS sunsets in March 2026](https://learn.microsoft.com/en-us/azure/aks/confidential-containers-overview); **VN2 is the recommended forward path.** |
| **Multi-tenant isolation** | Mutually distrusting tenants can share one AKS cluster, because each pod is its own micro-VM. Collapse N clusters into one without compromising the trust boundary. |

---

## VN2 in practice

Full cluster setup, manifests, Helm install, and network design (Azure CNI + delegated ACI subnet) are in the companion repo:

> **Demo repository:** *https://github.com/hailugebru/AKS-VN2/tree/main*

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

`kubectl describe`, `kubectl logs`, and `kubectl exec` work as they would on any node — including a shell into a pod running inside an ACI micro-VM.

<img width="960" height="322" alt="image" src="https://github.com/user-attachments/assets/3f5a6bf7-49d4-4850-927b-fc107f6c3c34" />

> *[Screenshot 1: `kubectl get` / `kubectl logs` / `kubectl exec` against a VN2-hosted pod.]*

Scaling stays trivial. `kubectl scale deployment demo-deployment --replicas=10` lands every replica on the same virtual node — no VMSS scale event, no provisioning latency, no climbing node-count chart. The same flow scales just as cleanly to hundreds.

<img width="1402" height="490" alt="image" src="https://github.com/user-attachments/assets/f7842539-7736-4d4c-89d0-9b87bff6e014" />

> *[Screenshot 2: `kubectl get pods -o wide` after scaling — every replica on the virtual node, no additional VMs.]*

### One annotation makes a pod confidential

The single switch that turns a regular VN2 pod into a confidential one is a pod annotation holding the CCE (Container Confidential Enforcement) policy — a base64 Rego document that pins exactly which images, commands, environment variables, mounts, and capabilities are permitted inside the encrypted micro-VM.

You don't write that policy by hand:

```bash
az extension add -n confcom
az confcom acipolicygen --virtual-node-yaml ./hello-world-deployment.yaml
```

The tool pulls each image, hashes its layers, builds the allow-list, and injects the annotation back into the manifest. `kubectl apply`, and you're done.

<img width="1578" height="418" alt="image" src="https://github.com/user-attachments/assets/fe844450-6423-4ad3-baff-b0f4c7c13925" />


> *[Screenshot 3: `az confcom acipolicygen` pulling and hashing images, emitting the base64 policy.]*

The detail that makes this a genuinely new isolation primitive: **the policy is enforced by the guest OS inside the encrypted boundary** — not by a control-plane admission webhook that an attacker on the host could bypass. The attestation report is retrievable from inside the pod, giving you a cryptographic *"prove to me you did not tamper with my workload"* guarantee that regulated industries — and increasingly, AI workloads running untrusted code — have been asking for. Background: [Microsoft Learn: Confidential Containers on ACI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-confidential-overview).

> *[Screenshot 4: Attestation report from inside the confidential pod, execution-policy hash highlighted.]*

---

## Where VN2 fits

Pair VN2 with traditional node pools, NAP, or Virtual Machine Node Pools for steady-state, DaemonSet, and persistent-volume workloads. **VN2 is additive, not a replacement.** For current capabilities, region availability, and known limitations, [Microsoft Learn is the source of truth](https://learn.microsoft.com/en-us/azure/aks/virtual-nodes).

---

**VN2 expands what AKS can run — more capacity, stronger isolation, per-second economics — without changing the Kubernetes operating model you already use.** Same `kubectl`, same manifests, same GitOps. New ceiling.
---
