# A New Chapter for AKS at Scale: Virtual Nodes on Azure Container Instances (VN2) — Effortless Capacity and Confidential Containers, Without Leaving Kubernetes

**Tags:** Azure Kubernetes Service, AKS, Azure Container Instances, ACI, Virtual Nodes, VN2, Confidential Containers, Serverless Kubernetes, Azure

*Co-authored with Gurpreet Virdi and the AKS / ACI engineering team.*

---

## A quick word before we begin

Azure Kubernetes Service is a powerful, fully managed Kubernetes platform that gives enterprises a rich open-source ecosystem and the operational flexibility to run almost anything — from line-of-business APIs to large-scale AI inference.

What this post is about is something different: **a newer capability inside AKS that is quietly changing what "running Kubernetes at scale" can feel like.** It's called **Virtual Nodes on Azure Container Instances**, or **VN2**, and it gives AKS clusters access to Azure's massive shared compute fleet through the same `kubectl`, the same Helm charts, the same GitOps pipelines you already use.

There are two reasons I wanted to write this post:

1. VN2 makes it remarkably easy to run hundreds of pods from a single virtual node — **no node pool sizing, no autoscaler wait, no idle-VM cost**.
2. VN2 is also the path to **Confidential Containers** on AKS — hardware-attested, per-container isolation that opens up entirely new categories of regulated and AI workloads.

I built an AKS cluster to demonstrate both, and the rest of this post walks through what I saw.

---

## What customers have been asking for

Working with enterprise customers every day, I've noticed a pattern in the conversations I have about AKS. The platform itself is loved. What teams keep asking for is *less of the work around it*:

- *"Can we stop guessing the right VM SKU for a node pool?"*
- *"Can we avoid the two-minute wait when the cluster autoscaler provisions a new VM under a traffic spike?"*
- *"Can we keep utilization high without manually tuning bin-packing and pod-density settings?"*
- *"Can we run multi-tenant workloads safely on one cluster instead of one cluster per tenant?"*
- *"Can we run a regulated or sovereign workload with hardware-rooted attestation — not just on a confidential VM, but per container?"*

These are good problems to have. They mean teams are succeeding *with* Kubernetes and now want to take it further.

### The capacity story underneath these questions

A lot of the operational nuance behind those conversations comes back to one thing: **a traditional node pool is tied to a specific VM SKU, region, and zone.** When demand spikes — or when Azure capacity in that exact SKU/region/zone is momentarily tight — that bond shows up as familiar allocation errors (`SkuNotAvailable`, `AllocationFailed`, `ZonalAllocationFailed`, quota exceeded), and the choice becomes overprovisioning "just in case" or underprovisioning and being stuck. The [AKS Engineering Blog covers this well](https://blog.aks.azure.com/2025/12/06/node-auto-provisioning-capacity-management).

AKS already has strong answers here: **Node Auto Provisioning (NAP)** picks the right VM SKU dynamically, and **Virtual Machine Node Pools** let a single pool span multiple SKUs. **VN2 is complementary** — for bursty, event-driven, or confidential workloads, it sidesteps the VM-allocation question entirely because the unit of capacity is no longer a VM. NAP and VM Node Pools handle steady-state; VN2 absorbs the spikes and the specialized isolation workloads on top.

---

## What VN2 is, in one sentence

> **VN2 lets your AKS cluster schedule pods onto Azure Container Instances as if ACI were a single, effectively unlimited Kubernetes node — no code changes, no manifest rewrites, no leaving the AKS world.**

Microsoft Learn describes the official capability this way: *"Virtual nodes on Azure Container Instances allow you to deploy pods in your Azure Kubernetes Service (AKS) cluster that run as container groups in ACI… It offers significantly better support for Kubernetes functionality, removing most of the limitations of the prior implementation."* ([Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes))

The key insight is that **ACI is not just a container service — it's a micro-VM platform.** Every container deployed to ACI runs in its own hardware-isolated micro-VM on a massive shared fleet (currently more than 1.1 million pooled cores) that Azure operates. Plug that fleet into AKS through a virtual node, and the picture changes:

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
   │ (small VM, just for │               │  (single logical node, │
   │  control-plane      │               │   backed by Azure's    │
   │  add-ons)           │               │   shared ACI fleet)    │
   └─────────────────────┘               └─────────┬──────────────┘
                                                   │
                                                   ▼
                                   ┌──────────────────────────────┐
                                   │   ACI serverless fleet       │
                                   │   (~1.1M cores, micro-VM     │
                                   │    isolation per pod)        │
                                   └──────────────────────────────┘
```

From the developer's perspective, **nothing changes**. The same `kubectl apply`, the same Helm charts, the same GitOps pipelines. The pod simply lands on a virtual node, and the virtual node hands it off to ACI behind the scenes.

> *[Screenshot placeholder: `kubectl get nodes` output showing the system node and the `virtual-node` entry, with `kubectl describe node virtual-node` highlighting the labels and taints.]*

---

## What this unlocks

### 1. Effortless capacity for bursty and event-driven workloads

In the cluster I built for this post, I can schedule **hundreds of pods against a single virtual node within seconds** — without provisioning a second VM, picking a SKU, or waiting on the cluster autoscaler. The virtual node never "fills up" the way a VMSS-backed node does, because the underlying capacity is the shared ACI fleet, not a fixed-size VM.

What that means in practice:

- **No `--max-pods` ceiling to negotiate** against the CNI's IP allocation.
- **No cold-start tax** while a fresh VMSS instance joins the cluster (which I have routinely seen take more than two minutes in real environments).
- **No node-pool right-sizing exercise** — there is no VM SKU to pick, because the VM is no longer the unit of capacity.

> *[Screenshot placeholder: `kubectl scale deployment myapp --replicas=200` followed by `kubectl get pods -o wide | grep virtual-node | wc -l` showing all 200 pods scheduled to the single virtual node in seconds.]*

> *[Screenshot placeholder: Azure portal — AKS Container Insights blade showing pod count vs. node count, with the pod curve climbing steeply while node count stays flat.]*

### 2. Per-second, per-pod billing

ACI bills **per second of actual pod execution time** — so for bursty workloads (batch jobs, CI agents, KEDA-scaled event handlers, AI inference) you pay for the pod-seconds you actually consumed. Used together with traditional node pools for steady-state workloads, this gives teams a cleaner FinOps story: predictable baseline capacity in your node pools, elastic burst capacity on VN2.

### 3. Confidential Containers — a genuinely new capability

This is the part I'm most excited about. Confidential ACI runs each container's micro-VM **encrypted with a hardware chipset key**, and produces a cryptographic **attestation report** that proves to you the micro-VM has not been tampered with — *including by Microsoft itself.* For sovereign cloud, regulated industries, and AI workloads running untrusted code, that's a meaningful step forward.

A quick note on the landscape today, because there are a few things in flight and it's worth being precise about them:

| | Confidential VMs (AMD SEV-SNP) on AKS | **Confidential Containers via VN2** |
|---|---|---|
| Trust boundary | The VM | **Each individual container (micro-VM per pod)** |
| Hardware-rooted attestation | VM-level | **Container-level, with verifiable execution policy** |
| Multi-tenant on the same host | Same trust domain | **Different trust domains can co-locate safely** |
| Workload shape | Lift-and-shift with VM-level isolation | **Lift-and-shift with per-pod isolation** |

It's worth noting the broader roadmap context: the standalone *Confidential Containers (preview) on AKS* — the Kata-on-CVM offering — [is set to sunset in March 2026, per Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/confidential-containers-overview), with **Confidential Containers on Azure Container Instances** (the VN2 path) called out as the recommended forward direction. If confidential workloads are on your roadmap, VN2 is where the investment is going.

> *[Screenshot placeholder: The same AKS cluster running two deployments side-by-side — one regular, one confidential — with the attestation report retrieved from the confidential pod.]*

### 4. Multi-tenant isolation, simplified

Because every pod runs in its own micro-VM, **mutually distrusting tenants can share a single AKS cluster** safely. That's a meaningful shift for multi-tenant SaaS teams who today are running one AKS cluster per tenant for isolation reasons. With VN2, one cluster can serve many tenants — without compromising the security boundary each customer expects.

---

## Why this matters now

The infrastructure underneath AKS is getting more capable, more elastic, and more secure — and VN2 is the door through which AKS customers can take advantage of that, *without changing how they build software.* The same `kubectl`, the same Helm charts, the same GitOps pipelines, with a much more elastic and isolation-capable compute layer underneath.

---

## The demo cluster

To make this concrete, I built an AKS cluster that demonstrates both capabilities end-to-end. The full setup is in the companion GitHub repo:

> **Demo repository:** *[GitHub link placeholder — e.g., `github.com/hailugebru/aks-vn2-capacity-confidential-demo`]*
> The README covers prerequisites, network design (Azure CNI + delegated subnet for ACI), Helm-based VN2 install, and the manifests used for the scaling and confidential demos.

### Cluster topology

| Component | Configuration |
|---|---|
| AKS cluster | 1 system node pool (small VM, control-plane add-ons only) |
| Networking | Azure CNI (required for VN2), dedicated ACI delegated subnet |
| Virtual node | Installed via Helm chart from `microsoft/virtualnodesOnAzureContainerInstances` |
| Workload nodes | **VN2 only** — pods scheduled directly to the virtual node |
| Confidential workload | Same cluster, same manifest, scheduled to a confidential ACI pod via node selector |

### Demo 1 — VN2 *is* Kubernetes: same manifests, same `kubectl`, same operational muscle memory

The point I want to make first is that **a virtual node is a Kubernetes node.** You can list it, describe it, see its labels and taints, and target it with all the usual ways Kubernetes lets you say *where* a pod should land. Nothing exotic, nothing proprietary.

Here is the manifest I used. The only VN2-specific lines are the `nodeSelector` and the `toleration` — everything else is plain Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
  labels:
    app: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - 'counter=1; while true; do echo "Hello, World! Counter: $counter"; counter=$((counter+1)); sleep 1; done'
        image: mcr.microsoft.com/azure-cli
        name: hello-world-counter
        resources:
          limits:
            cpu: "1"
            memory: 1G
          requests:
            cpu: 100m
            memory: 128Mi
      nodeSelector:
        virtualization: virtualnode2
        "kubernetes.io/os": linux
      tolerations:
      - effect: NoSchedule
        key: virtual-kubelet.io/provider
        operator: Exists
```

Once the deployment is up, every familiar `kubectl` command works exactly as it would on a VM-backed node:

```bash
# Confirm the pods are running on the virtual node
kubectl get pods -o wide -l app=demo

# Inspect status, events, scheduling decisions
kubectl describe pod <pod-name>

# Tail logs from a pod running on ACI
kubectl logs <pod-name>

# Shell into a pod running on ACI
kubectl exec <pod-name> -it -- /bin/bash
```

That last one is worth pausing on. The pod is running inside a micro-VM on the ACI fleet — and `kubectl exec` still drops you into a shell, just like it would on any other node. **The Kubernetes experience is preserved end-to-end.**

> *[Screenshot 1: `kubectl get pods` showing three demo pods in `Running` state, followed by `kubectl logs` and `kubectl exec` against a VN2-hosted pod, dropping into a root shell.]*

#### Scaling is trivial — and the cluster doesn't grow

Now the second half of the demo, which is the part that really lands with customers. One of the core benefits of a Kubernetes Deployment is that scaling up and down is a one-liner — Kubernetes handles the orchestration of creating or terminating pods automatically.

```bash
# Scale from 3 to 10 replicas
kubectl scale deployment demo-deployment --replicas=10

# Confirm every pod landed on the virtual node
kubectl get pods -o wide -l app=demo
```

> *[Screenshot 2: `kubectl get pods -o wide` after scaling to 10 replicas, with the `NODE` column showing every pod scheduled to the virtual node — no additional VMs provisioned.]*

The interesting part is what *doesn't* happen: no VMSS scale event, no node provisioning latency, no climbing node-count chart in AKS Insights. Every replica lands on the **same** virtual node, because the virtual node is backed by Azure's shared ACI fleet — and that fleet doesn't "fill up" the way a fixed-size VM does. The same flow scales just as cleanly to hundreds of replicas; the cluster never has to grow.

### Demo 2 — Confidential Containers, same cluster, same manifest

Confidential containers are a high-security offering from ACI that gives you a strong, cryptographically enforced guarantee about **what** is running and **what that image is allowed to do.** ([Microsoft Learn: Confidential Containers on ACI overview](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-confidential-overview))

The magic happens through a single pod annotation:

```
microsoft.containerinstance.virtualnode.ccepolicy
```

That annotation holds the **CCE (Container Confidential Enforcement) policy** — a base64-encoded Rego document that pins exactly which container images, commands, environment variables, mounts, and capabilities are allowed inside the confidential micro-VM. The policy is enforced at admission by the guest OS itself, *inside* the hardware-encrypted boundary.

You don't write the policy by hand. The `confcom` extension for the Azure CLI generates it from your existing YAML:

```bash
# One-time: install the confcom extension
az extension add -n confcom

# Generate the CCE policy and inject the annotation into the YAML
az confcom acipolicygen --virtual-node-yaml ./hello-world-deployment.yaml
```

Under the hood, `acipolicygen` pulls each container image, hashes its layers, builds the allow-list for the workload, and writes the resulting policy directly back into the manifest's `metadata.annotations` block. **No manual editing.**

> *[Screenshot 3: `az confcom acipolicygen --virtual-node-yaml .\hello-world-deployment.yaml` output — showing the tool pulling and hashing the container images and emitting the base64 security policy hash.]*

Here is what the same `demo-deployment` manifest from Demo 1 looks like *after* the policy has been injected — note that everything below `metadata.annotations` is unchanged from the original Kubernetes manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      annotations:
        microsoft.containerinstance.virtualnode.ccepolicy: cGFja2FnZSBwb2xpY3kKCmltcG9ydCBmdXR1cmUua2V5d29yZHMuZXZlcnkK...   # base64 CCE policy, truncated
      labels:
        app: demo
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - 'counter=1; while true; do echo "Hello, World! Counter: $counter"; counter=$((counter+1));
          sleep 1; done'
        image: mcr.microsoft.com/azure-cli
        name: hello-world-counter
        resources:
          limits:
            cpu: '1'
            memory: 1G
          requests:
            cpu: 100m
            memory: 128Mi
      nodeSelector:
        kubernetes.io/os: linux
        virtualization: virtualnode2
      tolerations:
      - effect: NoSchedule
        key: virtual-kubelet.io/provider
        operator: Exists
```

When you `kubectl apply` this, every replica lands in its own hardware-encrypted micro-VM on the ACI fleet, with the CCE policy enforced at boot. Any drift from the policy — an unexpected image, a different command, a mounted volume that wasn't approved — is rejected by the guest OS itself, *not* by a control-plane admission webhook that an attacker on the host could bypass. That is the part that makes this a genuinely new isolation primitive.

The attestation report is retrievable directly from inside the pod — the foundation for the kind of *"prove to me you did not tamper with my workload"* guarantee that regulated industries (and increasingly, AI workloads running untrusted code) are asking for.

> *[Screenshot 4: Attestation report output retrieved from inside the confidential pod, with the execution-policy hash highlighted to show it matches the value generated by `acipolicygen`.]*

---

## Who VN2 is great for

Like any capability, VN2 is a fit for some workloads more than others. Here's how I've been framing it with customers:

| Scenario | Why VN2 shines |
|---|---|
| Bursty / event-driven workloads | Scale to hundreds of pods in seconds, no autoscaler wait |
| KEDA-scaled microservices | Burst capacity that's billed per pod-second |
| Batch jobs and CI agents | Spin up, run to completion, tear down — pay only for what ran |
| AI inference on untrusted code | Per-pod micro-VM isolation, with confidential variant available |
| Multi-tenant SaaS | Collapse N clusters into 1, with hardware-rooted tenant isolation |
| Sovereign / regulated workloads | Hardware-attested confidential containers, alongside regular pods |
| Prototyping new services | Skip the node pool design exercise; deploy and iterate |

And it pairs naturally with traditional AKS node pools when you have steady-state workloads, DaemonSets that need to run on every host, or services that require persistent volumes tied to the node lifecycle. **VN2 is additive, not a replacement.** A well-designed cluster can use both — a node pool for baseline, a virtual node for burst and for confidential workloads.

For the current list of VN2 capabilities, region availability, and known limitations, the [official Microsoft Learn page is the source of truth](https://learn.microsoft.com/en-us/azure/aks/virtual-nodes).

---

## Three takeaways

1. **VN2 makes capacity feel effortless on AKS.** No VM SKU to pick, no autoscaler delay, no idle-VM cost for bursty workloads. The cluster simply schedules, and ACI handles the rest.
2. **Confidential Containers on VN2 are a meaningful new isolation primitive.** Per-container hardware attestation opens up sovereign, regulated, and AI-on-untrusted-code workloads on AKS that were previously hard to support.
3. **Stay on AKS — that's the whole point.** VN2 is not a migration away from Kubernetes. It is a way to *expand* what AKS can do: the same orchestration, the same ecosystem, the same tooling, with a much more elastic and isolation-capable compute layer underneath.

---

## Get started

- Try VN2 in a non-production AKS cluster this week. The [step-by-step deployment guide on Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) is the best starting point.
- Stand up one confidential ACI pod alongside a regular workload and verify the attestation report.
- Look at your portfolio for one multi-tenant or burst workload that would benefit, and prototype it on VN2.

If you try the demo cluster from the companion repo, I'd love to hear how it goes. Feedback from the field is exactly what helps the team prioritize what comes next.

---

## Acknowledgements

This post was developed in close collaboration with **Gurpreet Virdi** and the AKS / ACI engineering leadership team. Thank you for the architecture deep-dives, the candid roadmap context, and the willingness to bring the field along for the journey.

---

## Resources

| Resource | Link |
|---|---|
| Virtual Nodes on ACI — official docs | [learn.microsoft.com/azure/container-instances/container-instances-virtual-nodes](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-virtual-nodes) |
| AKS Virtual Nodes overview | [learn.microsoft.com/azure/aks/virtual-nodes](https://learn.microsoft.com/en-us/azure/aks/virtual-nodes) |
| AKS cost-optimization best practices | [learn.microsoft.com/azure/aks/best-practices-cost](https://learn.microsoft.com/en-us/azure/aks/best-practices-cost) |
| Confidential Containers on AKS — forward direction | [learn.microsoft.com/azure/aks/confidential-containers-overview](https://learn.microsoft.com/en-us/azure/aks/confidential-containers-overview) |
| VN2 Helm chart and customization | [microsoft.github.io/virtualnodesOnAzureContainerInstances](https://microsoft.github.io/virtualnodesOnAzureContainerInstances/) |
| Demo repository (this post) | *[GitHub link placeholder]* |

---

*Hailu Kassa is a Senior Cloud Solution Architect at Microsoft, focused on Azure Kubernetes Service and cloud-native architectures. Prior posts include [A Practical Guide to Autoscaling Web Applications with AKS, KEDA, and MSSQL](https://techcommunity.microsoft.com/t5/user/viewprofilepage/user-id/2436915) and [Autonomous AKS Incident Response with Azure SRE Agent](https://github.com/hailugebru/azure-sre-agents-aks).*
