# Heterogeneous Laptop Cluster Strategy

## Purpose

This repository supports a collection of heterogeneous laptops used as a private,
local AI appliance. The cluster should present a small number of stable interfaces
to users while allowing individual machines to serve different roles according to
their hardware, sustained performance, reliability, and economic value.

The strategy has four parts:

1. Treat nodes as reproducibly provisioned and replaceable.
2. Use cloud-init to establish the host and RKE2 membership contract, then let
   Kubernetes own application state.
3. Keep the laptop fleet in a dedicated cluster and integrate it with production
   through a narrow, authenticated model-service API.
4. Use heterogeneous capacity for a model refinery and complementary inference
   services instead of forcing every laptop into one latency-sensitive model.

The cluster is not intended to make mismatched GPUs appear as a cache-coherent
device or to preserve irreplaceable state on individual laptops.

## Architectural boundaries

| Layer | Owner | Examples |
| --- | --- | --- |
| Physical hardware | Human/vendor tooling | Firmware, BIOS/UEFI, cabling, power, cooling |
| Installation | iPXE and Ubuntu Autoinstall | Disk layout, base operating-system image |
| First-boot host contract | cloud-init | Identity, access, hardware profile, RKE2 configuration |
| Cluster infrastructure | RKE2 and GitOps | CNI, GPU integration, storage classes, observability |
| Model services | Kubernetes resources and operators | Model servers, batch jobs, routing, evaluation |
| User interface | API gateway/application | Stable local inference and agent endpoints |

Cloud-init is a first-boot mechanism, not an ongoing configuration-management
controller. Material host drift is corrected by reprovisioning. Emergency repairs
may be performed manually, but a repair that should survive reprovisioning must be
encoded in the installation image, hardware-class profile, or cloud-init data.

## Cluster isolation and production integration

### Dedicated-cluster decision

The laptop fleet runs as a dedicated RKE2 cluster. Laptop nodes do not join the
production `stable` Kubernetes control plane, and the two clusters do not share
etcd, CNI, storage, service accounts, admission webhooks, or cluster-scoped
operators.

Appropriate power, cooling, and wired networking make laptops better workers, but
do not make their firmware, batteries, consumer GPUs, kernels, or driver lifecycle
part of an acceptable production failure domain. A separate cluster also allows
GPU operators, Ray, experimental schedulers, and accelerated runtimes to evolve
without coupling their upgrade cadence to production.

The existing non-production clusters may host a short-lived canary node when an
integration must be tested against an existing platform. They are not the
permanent home of the laptop fleet.

### Integration boundary

```text
production applications
        |
        v
production LLM gateway
        |
        | private authenticated model API
        v
Beowulf ingress / inference gateway
        |
        +-- Ray Serve model services
        +-- other approved inference backends
        `-- model-refinery outputs promoted for serving
```

Production applications continue to use the production LLM gateway. That gateway
may route approved model aliases to an OpenAI-compatible endpoint exposed by the
Beowulf cluster over a private network. Authentication and encryption terminate at
the cluster boundary; internal model services are not exposed directly.

The boundary must ensure that:

- loss of every laptop removes only the corresponding local-model routes and does
  not impair unrelated production workloads;
- production retains an explicit timeout, circuit-breaker, and fallback policy;
- neither cluster receives administrator credentials for the other;
- only the credentials needed to invoke an approved endpoint cross the boundary;
- model artifacts and telemetry use narrowly scoped identities; and
- dashboards, Ray control APIs, Kubernetes APIs, and node services remain private.

### Recommended cluster topology

| Component | Placement | Responsibility |
| --- | --- | --- |
| Seed provisioning service | Independently bootstrapped management environment | iPXE, Ubuntu Autoinstall, and NoCloud data |
| RKE2 control plane and etcd | Three reliable, always-on systems or VMs | Dedicated Beowulf cluster state |
| Service worker pool | Reliable non-laptop nodes where available | Ingress, Ray head pods, operators, and supporting services |
| GPU worker pools | Qualified laptops grouped by compatible hardware class | Ray workers and model replicas |
| CPU worker pool | Qualified CPU/RAM-rich laptops | Data preparation, routing, indexing, and batch work |
| Artifact storage | Replicated service outside disposable node-local caches | Authoritative models, metadata, and refinery outputs |

Control-plane nodes do not run model workloads. If a non-laptop service-worker pool
is unavailable, the most reliable qualified machines may host Ray head and gateway
pods, but those roles remain isolated from GPU replicas and are replicated or
recoverable. Node-local NVMe is a cache, never the only copy of an artifact.

### Labels, taints, and admission policy

Every laptop worker is admitted with this baseline taint:

```text
beowulf.bcit.ca/laptop=true:NoSchedule
```

Only workloads designed to tolerate loss of a laptop may tolerate it. A toleration
makes a workload eligible for a tainted node; it does not select the correct node.
Model and refinery workloads therefore require both an explicit toleration and
required node affinity for an approved hardware class.

Hardware and policy are represented separately:

- trusted administrative labels record hardware class, reliability tier, and
  admitted workload roles;
- Node Feature Discovery and GPU components report observed runtime capabilities;
- Kubernetes extended resources represent consumable devices such as GPUs; and
- Ray worker groups expose only the resources provided by their selected nodes.

Security-sensitive placement labels use a prefix protected by the Kubernetes
`NodeRestriction` admission plugin, for example:

```text
beowulf.bcit.ca.node-restriction.kubernetes.io/hardware-class=nvidia-modern
beowulf.bcit.ca.node-restriction.kubernetes.io/reliability-tier=qualified
beowulf.bcit.ca.node-restriction.kubernetes.io/workload-role=inference
```

Cluster bootstrap validation must confirm that the Node authorizer and
`NodeRestriction` admission plugin enforce this protection before these labels are
used for workload isolation.

Additional quarantine or experimental taints may prevent new scheduling, but node
fault handling remains an explicit cordon-and-drain procedure. A custom
`NoExecute` taint must not be applied automatically until its eviction and storage
consequences have been tested. Production namespaces must never contain broad
`Exists` tolerations for project taints.

### NVIDIA GPU enablement on RKE2

The NVIDIA GPU Operator is the cluster-level integration for admitted NVIDIA
workers. It owns the Kubernetes-facing GPU stack: container-runtime integration,
the NVIDIA device plugin, GPU Feature Discovery, validation, and DCGM-based
monitoring. Ray workloads consume the resulting `nvidia.com/gpu` extended resource;
they do not install or reconfigure the runtime themselves.

Kernel-driver ownership is selected once per hardware-class profile:

| Driver mode | Provisioning responsibility | GPU Operator configuration |
| --- | --- | --- |
| Operator-managed | cloud-init supplies kernel prerequisites; the operator installs the validated driver | Driver operand enabled |
| Host-managed | cloud-init installs and pins a validated mobile/legacy driver | Driver operand disabled; other operands remain enabled |

The two modes must not manage the driver simultaneously. Host-managed mode remains
available because consumer laptop GPUs and older hardware may require a driver that
is not compatible with the operator's default driver container. A node class is
admitted only after its exact Ubuntu, kernel, driver, RKE2/containerd, GPU Operator,
and CUDA interface combination passes validation.

The deployment must follow the RKE2-specific integration contract:

- configure the NVIDIA Container Toolkit with the RKE2 containerd socket at
  `/run/k3s/containerd/containerd.sock`;
- pin the GPU Operator chart and operand versions through GitOps;
- select CDI or the NVIDIA runtime class according to the validated RKE2,
  containerd, and GPU Operator versions rather than mixing both paths implicitly;
- modify the RKE2 service `PATH` only when required, using trusted, explicitly
  declared directories;
- keep experimental NRI integration disabled for the initial implementation;
- ensure CPU, AMD, unsupported NVIDIA, and quarantined nodes are excluded from
  NVIDIA operands, using controls such as
  `nvidia.com/gpu.deploy.operands=false`; and
- coordinate Node Feature Discovery ownership so it is installed exactly once.

GPU Operator runtime changes can restart RKE2 on a node. Installation and upgrades
therefore proceed one hardware class and one node at a time after cordon and drain,
with capacity and rollback verified before continuing. An operator or driver update
must not roll across all inference replicas simultaneously.

An NVIDIA node becomes eligible for Ray only after automated checks confirm:

1. the expected `nvidia.com/*` discovery labels are present;
2. `nvidia.com/gpu` reports the expected allocatable count;
3. the NVIDIA container runtime or CDI configuration is active;
4. the GPU Operator validators are healthy;
5. a pinned CUDA sample can request and exercise the GPU; and
6. DCGM metrics and the node's thermal telemetry are visible.

## Node lifecycle

```text
unregistered
    -> discovered
    -> assessed
    -> disposition approved
    -> provisioning
    -> ready
    -> active
    -> drained
    -> reprovisioning -> ready
                    or -> retired
```

### 1. Unregistered

The laptop has not been admitted to the fleet. No assumptions are made about its
condition, installed operating system, storage contents, firmware, or suitability.

### 2. Discovered

The device receives a stable fleet identity based on recorded asset information,
including manufacturer, model, serial number, network identities, and relevant PCI
devices. A MAC address may select a boot profile but is not the sole durable device
identity.

### 3. Assessed

The inventory and assessment process in
[#1](https://github.com/bcit-tlu/beowulf-cluster/issues/1) records raw hardware,
compatibility, performance, sustained thermal, network, storage, and optional power
measurements. Results are versioned and remain separate from policy judgments.

The assessment assigns factual capabilities such as:

- inference backend support;
- usable VRAM class;
- sustained performance class;
- wired network class;
- storage and CPU/RAM capabilities; and
- reliability or thermal limitations.

### 4. Disposition approved

The evaluation described in
[#3](https://github.com/bcit-tlu/beowulf-cluster/issues/3) recommends whether the
device should be retained for one or more roles, harvested for parts, or sold. The
recommendation must identify measured facts, assumptions, missing evidence, and
policy weights. It never initiates disposal automatically.

### 5. Provisioning

An independently available seed service supplies iPXE, Ubuntu Autoinstall, and
NoCloud data. Provisioning is versioned and repeatable. The seed service must not
depend on a cluster that is still being created; it runs on an existing management
node, a seed cluster, or another independently bootstrapped environment. Its design
is tracked in [#2](https://github.com/bcit-tlu/beowulf-cluster/issues/2).

### 6. Ready

The laptop has completed first boot, joined RKE2, passed host and GPU validation,
and received its declared labels and taints. It is not yet assumed to be carrying a
production workload.

### 7. Active

GitOps-managed cluster resources assign work that matches the node's capabilities.
Replaceable model files and caches may reside on local NVMe; authoritative metadata,
configuration, and evaluation results must not exist only on one laptop.

### 8. Drained

Workloads are evicted or completed before maintenance, reprovisioning, reassessment,
or retirement. A node with thermal, storage, GPU, or power faults is cordoned and
drained rather than allowed to degrade a synchronized workload.

### 9. Reprovisioned or retired

Host drift, incompatible driver changes, and unexplained configuration damage are
normally resolved by returning to the provisioning state. A device that no longer
has a defensible role returns to disposition review and may be retired.

## Provisioning with Ubuntu Autoinstall and NoCloud

### Seed prerequisites

Before enrolling laptops, provide:

- wired DHCP and DNS appropriate to the provisioning network;
- an iPXE-compatible boot path for supported UEFI devices;
- HTTP hosting for pinned Ubuntu installer artifacts and checksums;
- a NoCloud endpoint capable of returning device- or class-specific data;
- a secure mechanism for issuing RKE2 enrollment credentials; and
- an existing management environment in which the provisioning service can run.

Most laptops do not have a BMC. Network boot and installation can be automated, but
power control, firmware setup, and boot-device selection may require physical action.

### Hardware-class profiles

Profiles should describe only differences that must exist below Kubernetes. Begin
with a small set and split them only when the hardware requires it:

- RKE2 server;
- CPU-only RKE2 agent;
- current-generation NVIDIA agent;
- legacy NVIDIA agent; and
- AMD/ROCm agent.

Each profile pins its Ubuntu release, kernel and driver policy, RKE2 release, and
installation artifact checksums. GPU kernel drivers may be installed in the host or
managed by a Kubernetes operator, but responsibility must be explicit and must not
be duplicated for the same hardware class.

### Provisioning sequence

1. **Select the node.** Record or confirm the durable fleet identity and approved
   hardware class.
2. **Network boot.** The laptop performs a UEFI PXE/iPXE boot over wired Ethernet.
3. **Resolve the profile.** The boot service maps the device to an approved Ubuntu,
   hardware, and RKE2 role profile. Unknown devices receive a diagnostic or
   assessment environment, not a default destructive installation.
4. **Load the installer.** iPXE loads pinned Ubuntu kernel and initrd artifacts and
   passes the Autoinstall and NoCloud data-source location.
5. **Install Ubuntu.** Autoinstall applies the declared disk layout and base image.
   Destructive storage changes occur only after the device has been positively
   identified and approved.
6. **Apply first-boot configuration.** NoCloud `meta-data`, `user-data`, and optional
   `vendor-data` establish hostname, SSH access, time synchronization, package
   sources, required host settings, driver policy, and RKE2 configuration.
7. **Enroll in RKE2.** The node obtains a short-lived or otherwise protected join
   credential, configures its server or agent role, and joins the intended cluster.
8. **Declare static capabilities.** Bootstrap configuration applies only trusted,
   static labels and taints such as hardware class or intended control-plane role.
   Runtime GPU and PCI facts are discovered and verified by cluster components.
9. **Reconcile cluster services.** GitOps installs GPU device plugins/operators,
   feature discovery, monitoring, model runtimes, caches, and workload resources.
10. **Validate readiness.** Automated checks confirm node health, expected GPU
    access, local storage, wired networking, sustained thermals, and a minimal
    inference workload before the node becomes active.

### Security requirements

- Treat NoCloud data and boot URLs as potentially observable on the provisioning
  network.
- Do not commit reusable RKE2 tokens, SSH private keys, API credentials, or model
  repository credentials.
- Prefer per-node, short-lived enrollment material and revoke it after successful
  admission.
- Pin and verify installation artifacts.
- Isolate provisioning traffic from untrusted networks.
- Ensure an unknown device cannot select a destructive installation profile merely
  by presenting an unrecognized MAC address.

### Reproducibility and change control

Provisioning inputs are reviewed and versioned together:

- installer and kernel versions;
- hardware-class profile;
- cloud-init documents;
- RKE2 version and configuration;
- driver ownership and version policy; and
- validation suite version.

A profile change is tested on a representative node before broader reprovisioning.
Rollback means redeploying the last known-good profile rather than reversing a list
of in-place mutations.

## Model refinery

### Goal

The model refinery converts heterogeneous, independently useful compute into better
local AI artifacts and evidence. It emphasizes parallel work with infrequent
coordination so that weaker nodes contribute without becoming part of every token's
latency-critical path.

The refinery complements live inference. It is not itself the user-facing chat or
agent interface.

### Refinery workflow

```text
source corpora and tasks
        -> normalize, filter, and index
        -> generate candidate data and responses in parallel
        -> critique, rank, and verify
        -> evaluate models, prompts, adapters, and quantizations
        -> promote approved artifacts
        -> serve through the cluster's stable local API
```

Candidate refinery workloads include:

- synthetic instruction and test-data generation;
- multiple independent candidate solutions;
- coding, reasoning, factuality, safety, and style critique;
- retrieval-corpus parsing, chunking, embedding, and indexing;
- regression suites across models, prompts, and quantizations;
- LoRA data preparation and, where supported, training;
- quantization and conversion;
- long-running thermal and performance qualification; and
- shadow evaluation of proposed model-service changes.

### Mapping heterogeneous hardware to work

| Capability | Preferred work |
| --- | --- |
| Highest VRAM and bandwidth | Primary reasoning models, target/verifier models, larger training or evaluation stages |
| Medium GPU | Coding specialists, draft models, candidate generation, reranking |
| Small or older GPU | Embeddings, OCR, speech, classifiers, small critics, backend-specific tests |
| CPU/RAM rich | Data preparation, tokenization, indexing, vector databases, orchestration |
| Fast local NVMe | Replaceable model and dataset caches, intermediate artifacts |
| Unstable or thermally limited | Short bounded jobs only, or disposition review |

Several compatible high-value nodes may form a deliberately tested distributed
model group. That group is exposed as one backend to the gateway; it does not define
the architecture of the entire fleet.

### Execution and artifact principles

- Use Kubernetes Jobs or an appropriate batch controller for finite refinery stages.
- Make each stage restartable and content-address its inputs and outputs.
- Record model, prompt, sampler, seed, runtime, driver, benchmark, and policy versions.
- Separate generated candidates from accepted training or evaluation artifacts.
- Promote artifacts only after declared evaluation gates pass.
- Store authoritative results outside ephemeral node-local caches.
- Measure useful output, quality, latency, sustained throughput, and energy where
  possible; raw token throughput alone is not a promotion criterion.

### Single-interface presentation

Users and applications interact with a stable local API. The gateway may route a
request to:

- a replicated independent model;
- a coding or reasoning specialist;
- a draft/target speculative pair;
- retrieval, embedding, vision, or speech services; or
- a distributed large-model backend composed of a compatible subset of nodes.

Agent roles such as coder, reasoner, and critic belong to the application workflow.
Kubernetes resources describe and operate model capabilities; they should not encode
an entire conversation graph into host provisioning.

## Initial implementation order

1. Complete the fleet inventory and assessment work in #1.
2. Define disposition policy in #3 and select the initial retained fleet.
3. Implement and validate the seed iPXE/NoCloud service in #2.
4. Provision a small RKE2 cluster from representative hardware classes.
5. Establish GPU discovery, node classification, monitoring, and local model caches.
6. Install the KubeRay operator and expose one independent model with a
   `RayService`, following the [Ray Serve implementation requirements](./RAY_SERVE.md).
7. Add one refinery workflow with reproducible inputs, outputs, and evaluation.
8. Evaluate distributed inference only on compatible subsets and retain it only when
   it improves a declared capacity, latency, throughput, or research objective.

## References

- [cloud-init user-data formats](https://cloudinit.readthedocs.io/topics/format.html)
- [cloud-init module frequencies](https://cloudinit.readthedocs.io/en/latest/topics/modules.html)
- [RKE2 documentation](https://docs.rke2.io/)
- [RKE2 GPU Operators](https://docs.rke2.io/add-ons/gpu_operators)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [KServe](https://kserve.github.io/website/)
- [KubeRay](https://ray-project.github.io/kuberay/)
- [Ray Serve implementation requirements](./RAY_SERVE.md)
