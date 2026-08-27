# Virtual nodes on Azure Container Instances: A Powerful New Way to Run Containers on Azure

*Effortless burst capacity, confidential containers, and a serverless compute layer for Kubernetes on Azure.*

## Meet virtual nodes on ACI

If you package your applications as containers, you eventually face the same question: what do you run them on? Kubernetes is the industry standard answer, and Azure Kubernetes Service (AKS) is the managed version of it, where Microsoft operates the control plane and you bring the applications. The part that stays your job is capacity. A Kubernetes cluster normally runs your containers on a pool of virtual machines that you size, scale, patch, and pay for whether or not they are busy.

Virtual nodes on Azure Container Instances remove that job for the workloads that need it least. They let your containers run directly on Azure's serverless container platform, billed by the second for the cores and memory actually used, with no virtual machines to size or manage.

**Where you are probably starting from:**

* **New to containers on Azure.** You have applications you want to host, and you are weighing what to run them on. Virtual nodes on ACI mean you can start with a small managed cluster and let Azure absorb the peaks, instead of sizing infrastructure for a load you have not measured yet.
* **Already running AKS.** You know the capacity tradeoff well. This gives you a place to send spikes, short lived jobs, and workloads needing stronger isolation, without disturbing what you run today.
* **Evaluated the original virtual nodes and walked away.** Read the next section carefully. The reasons you walked away are largely gone.

In short: virtual nodes on ACI attach Azure's serverless container platform to a Kubernetes cluster as if it were a node. Containers scheduled there run as Hyper-V isolated containers, up to 200 per virtual node. Run multiple virtual nodes for more. Everything you already use to manage a cluster, whether that is `kubectl`, Helm charts, or a GitOps pipeline, works exactly the same way.

> **[PLACEHOLDER: per pod limits]** Replace the draft's "effectively unlimited cores and memory" with the real per pod vCPU and memory ceiling, and note that regional quota applies. Unbounded claims are the fastest way to lose a technical reader, and a reviewer will flag it.

Two things stand out.

**Effortless burst capacity.** Schedule up to 200 containers per virtual node in seconds, and run multiple virtual nodes for more, with no capacity planning and no waiting for machines to come online. You pay per second for the cores and memory used, so idle capacity costs nothing.

**Confidential containers.** Hardware attested, per container isolation inside a Trusted Execution Environment (TEE). A TEE is a protected region of the processor that even the host operating system cannot inspect, which opens up regulated, sovereign, and AI on untrusted code workloads. The same per container isolation also makes it practical to run multiple tenants on a single cluster, provided you add the network boundaries that tenant isolation requires.

> **[VERIFY]** Link the current public guidance on those network boundaries, or cut the multi tenancy sentence. The draft's "which Microsoft is documenting" raises the benefit and withdraws it in the same breath, which is worse than not raising it.

### If you evaluated virtual nodes before, this is not a rebrand

There was an earlier AKS virtual nodes feature, built on Virtual Kubelet. Many teams tried it, hit a limitation that mattered to them, and moved on. This is a new implementation that integrates far more deeply with Kubernetes and lifts most of those limitations.

What changed:

* **Init containers.** Previously unsupported. Now supported.
* **Persistent volumes.** Previously unsupported. Now supported.
* **Managed identity.** Previously limited. Now supported.
* **Networking.** Previously constrained. Now a richer networking model.
* **Confidential containers.** Previously unavailable. Now built in.
* **Kubernetes integration.** Previously a thin shim. Now deeply integrated.

If none of that is familiar, you are not missing context you need. Skip ahead; nothing later in this post depends on it.

> **[VERIFY]** Confirm every line above against current public documentation before publishing, and add anything missing (DaemonSets, host networking, hostPath, probes). This list is load bearing. It is the thing that changes the mind of a reader who evaluated the original and gave up.

The migration guide on the Apps on Azure blog has the full detail.

## The capacity problem this solves

Every platform that runs containers on virtual machines inherits the same constraint, and it is worth understanding before you choose an architecture rather than after.

Machines come in fixed sizes, in specific regions, in specific availability zones. You reserve them in advance. When demand spikes, or when that exact machine size is momentarily scarce in that exact region and zone, capacity requests start failing. On Azure those failures have names you will meet in the portal and in logs: `SkuNotAvailable`, `AllocationFailed`, `ZonalAllocationFailed`, quota exceeded.

> **[PLACEHOLDER: customer proof point]** Insert one anonymized scenario here, in three sentences. Industry, workload shape, and the number that hurt. Suggested structure: *"A [industry] customer running [workload] held roughly [X] percent idle headroom year round to survive a [duration] spike."* This section currently contains no customer. It is the highest value addition in the post.

That leaves two options, and both cost you something. Overprovision, and you pay year round for capacity you need a few hours a month. Stay lean, and you risk not scaling on the day it matters.

**If you are already on AKS,** you have tools for part of this. Node Auto Provisioning picks machine sizes dynamically, and Virtual Machine Node Pools span multiple sizes, both of which widen the surface for steady state workloads. Details are on the AKS Engineering Blog.

**If you are choosing an architecture now,** the useful framing is that these are complementary layers rather than competing products. A pool of virtual machines is the right home for workloads that run continuously, because reserved capacity is cheaper per hour when you actually use every hour. Virtual nodes on ACI are the right home for demand you cannot predict, work that finishes quickly, and workloads that need stronger isolation, because the containers land on Azure's serverless platform directly and never wait for a machine to be allocated.

You do not have to choose one. A single cluster runs both, and the rest of this post shows how.

## How virtual nodes on ACI work

ACI runs every container as a Hyper-V isolated container, which means each one gets its own lightweight virtual machine boundary rather than sharing a kernel with its neighbors. Azure operates that platform. A virtual node connects it to your cluster.

The cluster's control plane, the component that decides where each container runs, sees two kinds of destination: a small pool of virtual machines carrying cluster services, and one or more virtual nodes. From your application's perspective, nothing changes.

![Architecture diagram. On the left, the Azure Kubernetes Service panel shows the AKS control plane scheduling to a system node pool and to a virtual node, with the virtual node inside a delegated subnet and targeted by a nodeSelector. On the right, the Azure Container Instances panel shows the serverless platform running one Hyper-V isolated container per pod, up to 200 pods per virtual node, with per second billing, hardware attested isolation available per container, and no node pool sizing or autoscaler delay.](aks_virtual_nodes_diagram_compact.png)

Two steps carry the whole model. The scheduler places the container on the virtual node, matched by the same routing rules it would use for any node. The virtual node then hands it to ACI, which runs it as a Hyper-V isolated container. Note the delegated subnet around the virtual node: that is a deployment requirement, not a detail, and the setup section returns to it.

> **[PLACEHOLDER: measured results]** Close this section with the numbers you measured. Time from request to running container on a virtual node versus a cold machine pool scale out, at 10 containers and at 100, plus the count of additional virtual machines provisioned in each case, which should be zero on the virtual node. State the test conditions: region, machine size, image size, and whether the image was cached. If you did not test at 100, publish only what you ran. A smaller honest number is worth more than an extrapolation, and this paragraph is what separates a walkthrough from a post that gets cited.

See Microsoft Learn: virtual nodes on ACI for the official capability description and current limits.

## Virtual nodes on ACI in practice

The rest of this post is hands on. If you have never used Kubernetes, you can still follow it. `kubectl` is the command line tool for talking to a cluster, Helm installs packaged software into one, and a manifest is a text file describing what you want to run. If you have a cluster, everything below runs against it as written.

The manifests behind the examples live in a companion demo repo. Setup is documented officially, and you can reproduce this end to end from the ACI virtual nodes documentation and the `microsoft/virtualnodesOnAzureContainerInstances` Helm repo.

One requirement before you start: deploy into a delegated ACI subnet, meaning a subnet in your virtual network set aside for the ACI platform to place containers in.

> **Warning: pin your AKS auto upgrade channel to patch only within a minor version.**
> Minor version jumps can ship breaking changes that regress virtual nodes until the Helm chart is realigned. If auto upgrade crosses a minor version, you can lose virtual node scheduling with no obvious cause. Treat this as a production prerequisite, not a footnote.

Demo manifests: https://github.com/hailugebru/AKS-VN2/tree/main

### Enable virtual nodes on ACI

The virtual node is deployed via Helm. The Microsoft GitHub repo is itself a Helm repository, so a single `helm install` is all you strictly need. Cloning first, shown here, just makes it easier to customize values. Running `kubectl get nodes` afterward confirms the node registered.

```bash
git clone https://github.com/microsoft/virtualnodesOnAzureContainerInstances.git
helm install <yourReleaseName> ./virtualnodesOnAzureContainerInstances/Helm/virtualnode
kubectl get nodes
```

The virtual node appears alongside any existing capacity, ready to accept work.

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

Scaling stays trivial:

```bash
kubectl scale deployment demo-deployment --replicas=10
```

Every copy lands on the same virtual node. No machines are created, nothing waits for infrastructure, and the node count never moves.

> **[SCREENSHOT]** `kubectl get pods -o wide` after scaling, showing every replica on the virtual node and no additional VMs. This is one of only two screenshots worth keeping from the draft. Drop the `kubectl get nodes` shot and the logs and exec shot, which prove nothing the code blocks above already show.

> **[VERIFY]** The draft says this "scales just as cleanly to hundreds." Keep that sentence only if you ran it at that size. Otherwise cut it, because it is exactly the kind of claim a skeptical reader discounts, and discounting it costs you the rest of the post.

### One annotation makes a container confidential

Turning a regular container into a confidential one takes a single addition to its manifest: a policy that pins exactly which images, commands, environment variables, mounts, and capabilities are permitted inside the Trusted Execution Environment. The format is a base64 encoded Rego document, called a CCE (Container Confidential Enforcement) policy.

You do not write that policy by hand. A tool generates it from the manifest you already have:

```bash
az extension add -n confcom
az confcom acipolicygen --virtual-node-yaml ./hello-world-deployment.yaml
```

The tool pulls each image, hashes its layers, builds the allow list, and writes the policy back into the manifest. Run `kubectl apply`, and you are done. (`acipolicygen` has prerequisites of its own, including a working Docker installation. See the confcom documentation.)

> **[SCREENSHOT]** `az confcom acipolicygen` pulling and hashing images, emitting the base64 policy.

Here is why this is a genuinely new isolation primitive rather than a stronger version of an existing one. Most container security policy is enforced by software in the cluster, which means an attacker who compromises the host can potentially bypass it. This policy is enforced by the guest operating system inside the TEE instead. The hardware also produces an attestation report, retrievable from inside the container, which is a cryptographic proof that the workload running is the workload you specified and nothing tampered with it. That is the guarantee regulated industries have been asking for, and increasingly the one AI workloads running untrusted code need too.

> **[PLACEHOLDER: attestation demo]** Show the report. The command run from inside the container, the redacted output, and one sentence on what a verifier does with it. This is the strongest technical claim in the post and currently the only major one with no evidence behind it. It is the screenshot worth having.

Background: Microsoft Learn: confidential containers on ACI.

## Wrapping up

Virtual nodes on ACI give containers on Azure two things that were previously hard to deliver cleanly on Kubernetes:

* **Effortless burst capacity** on Azure's serverless container platform, billed per second for the cores and memory used, with no capacity planning and no waiting for machines.
* **Confidential containers** with hardware attested, per container isolation inside a Trusted Execution Environment.

Both come with a boundary worth naming. Virtual nodes on ACI are the wrong tool for:

* **Steady state workloads.** Per second billing wins on bursty and short lived work. Reserved virtual machines win on anything that runs around the clock. Do the math for your own shape before you move anything.
* **Cluster wide agents** that need to run a copy on every machine, such as log collectors and monitoring daemons. There is no machine for them to run on.
* **Workloads needing direct access to the host** network, filesystem, or privileged operations.
* **Latency sensitive request paths** where container start time sits on the critical path.
* **Multi tenancy without additional network boundaries.** Per container isolation is a strong foundation, but it is not the whole story.

> **[PLACEHOLDER: cost break even]** Add one concrete comparison: the cost of N vCPU hours on ACI versus the equivalent reserved or pay as you go virtual machine, and the approximate utilization point where the two cross. The post mentions per second billing three times without ever comparing it to a virtual machine, which leaves your strongest argument unmade.

It is additive, not a replacement. Pools of virtual machines remain the right home for continuous, agent based, and storage attached workloads. Virtual nodes on ACI absorb the spikes, the short lived jobs, and the specialized isolation work on top.

### Where to start

* **New to containers on Azure?** Start with a small AKS cluster and add a virtual node from day one. You get a managed Kubernetes environment without having to guess your peak capacity in advance, and the elastic layer is there the first time you need it.
* **Already running AKS?** Add a virtual node to an existing cluster and move one bursty or short lived workload to it. Nothing else changes, and the comparison is immediate.
* **Evaluating platforms?** The capability that is hard to find elsewhere is the confidential containers path: hardware attested isolation per container, reachable through a standard Kubernetes manifest.

The result: virtual nodes on ACI expand what a Kubernetes cluster on Azure can run, with more capacity, stronger isolation, and per second economics, without changing the operating model, whether you have been using it for years or are adopting it now.

Same kubectl, same manifests, same GitOps. New ceiling.

*For the high level overview, official documentation, and Helm details, the migration guide on the Apps on Azure blog and Microsoft Learn are the sources of truth. The companion repo holds the demo manifests used in this post.*
