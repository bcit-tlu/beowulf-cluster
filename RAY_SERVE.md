# Ray Serve Implementation Requirements

Status: draft for [issue #4](https://github.com/bcit-tlu/beowulf-cluster/issues/4)

## Decision summary

KubeRay and Ray Serve are the first model-serving runtime for the dedicated
Beowulf Kubernetes cluster. They are not the host provisioner: iPXE, Ubuntu
Autoinstall, cloud-init, and RKE2 create the cluster before KubeRay is installed.

The initial implementation uses the KubeRay operator and a `RayService` custom
resource. It proves one independently useful model on one qualified GPU before
attempting horizontal replicas, heterogeneous model routing, or cross-node model
parallelism. Versions are pinned only after the compatibility spike validates the
selected Ubuntu, kernel, GPU driver, container runtime, Ray, KubeRay, and inference
engine combination.

This is an explicit operator-first decision:

- use `RayService` for persistent Ray Serve inference applications;
- use `RayJob` for finite refinery stages when Ray is the appropriate executor;
- avoid CLI-managed production Serve state; and
- use a raw `RayCluster` only for diagnostics or a requirement that the higher-level
  resources cannot express.

## Responsibility boundaries

| Concern | Owner |
| --- | --- |
| Firmware, power, cooling, and cabling | Hardware operations |
| Ubuntu installation and first boot | iPXE, Autoinstall, and cloud-init |
| Kubernetes membership and node contract | RKE2 |
| NVIDIA runtime, device discovery, validation, and metrics | NVIDIA GPU Operator |
| Desired cluster state | GitOps |
| Ray cluster and Serve application lifecycle | KubeRay `RayService` controller |
| Replica placement within Ray capacity | Ray scheduler and placement groups |
| Model execution | Ray Serve LLM engine or another validated Serve deployment |
| Production authentication, aliases, fallback, and policy | Production LLM gateway |

Kubernetes first selects a compatible Ray worker pod using node affinity, taints,
tolerations, and device resources. Ray then places actors within the resources
advertised by those worker pods. Both scheduling layers must agree; a Ray custom
resource must not be used to bypass the Kubernetes placement contract.

## Initial architecture

```text
GitOps repository
    |
    +-- KubeRay operator
    `-- RayService
          |
          +-- Ray head pod on reliable service capacity
          +-- CPU worker group
          +-- one worker group per compatible GPU class
          `-- Ray Serve application
                  |
                  `-- validated inference engine and model

private production gateway --> Beowulf ingress --> Ray Serve HTTP service
model artifact store ----------------------------> node-local read-through cache
metrics and logs --------------------------------> cluster observability
```

The Ray head pod and KubeRay operator should run on reliable service capacity, not
on a GPU worker expected to be drained or power-cycled. GPU worker groups are
homogeneous enough that every replica in a group can use the same image, driver
interface, engine configuration, and model artifact.

## Scheduling contract

### Kubernetes layer

All laptop-targeted Ray workers require:

- toleration for `beowulf.bcit.ca/laptop=true:NoSchedule`;
- required affinity for an admitted hardware-class label;
- a GPU resource request when the worker requires a GPU;
- CPU, memory, and ephemeral-storage requests and limits based on measurement;
- topology rules that prevent replicas intended for redundancy from sharing one
  physical node; and
- a bounded termination grace period that permits request draining.

Worker groups must select classes, not individual hostnames. Proposed initial
classes are `cpu`, `nvidia-modern`, `nvidia-legacy`, and `amd-rocm`; inventory data
may split these further by VRAM or runtime compatibility. A class is admitted only
after its image and minimal inference workload pass validation.

### Ray layer

Each worker group advertises only measured resources that are actually available
inside its pod. Deployments declare CPU, GPU, memory, and any project-specific Ray
resource needed to distinguish incompatible accelerator classes.

The first deployment uses one replica and one GPU on one node. Later stages may:

1. add replicas within the same hardware class;
2. add independent specialist models on different classes; and
3. test tensor or pipeline parallelism on a compatible subset.

Cross-node parallelism is not a default. It is accepted only when the model cannot
fit on one node or measurements show a useful improvement after network overhead,
tail latency, failure coupling, and thermal throttling are included.

## Functional requirements

### RAY-FR-001: Declarative installation

The KubeRay operator, namespaces, RBAC, network policy, Ray service, configuration,
and monitoring resources are reconciled through GitOps. No successful deployment
may depend on an undocumented imperative command.

### RAY-FR-002: Reproducible runtime

Images are built from reviewed definitions and pinned by immutable digest. The
compatibility record includes Ubuntu, kernel, host driver, container runtime, GPU
runtime, Ray, KubeRay, Python, inference engine, and model format versions.

### RAY-FR-003: Model artifact handling

Model artifacts have a declared source, revision, checksum, license record, and
minimum resource class. Authoritative artifacts live in replicated storage.
Node-local NVMe acts as a replaceable read-through cache. Repository or object-store
credentials are injected at runtime and never embedded in images or manifests.

### RAY-FR-004: Stable inference interface

The first service exposes a health endpoint and an OpenAI-compatible inference API.
Only the Beowulf ingress is reachable across the cluster boundary. The production
gateway owns public model aliases, client authentication, request policy, timeouts,
fallbacks, and rate limits.

### RAY-FR-005: Controlled scaling

The proof of concept starts with fixed replica and worker counts. Ray Serve
autoscaling and KubeRay in-tree autoscaling are enabled only after queue behavior,
model load time, cache pressure, and worker-pod placement have been measured.
Autoscaling maxima must reflect physical fleet capacity; it cannot create a laptop
that is powered off or unavailable.

### RAY-FR-006: Failure recovery

Ray Serve health checks replace failed replicas, and `RayService` reports application
and cluster health. The implementation documents expected behavior for worker loss,
head-pod loss, node drain, model-load failure, and loss of artifact storage.

The initial service may accept a maintenance-window restart after Ray head failure.
External Redis-backed GCS fault tolerance is a later decision, not an implicit
dependency. No request path may assume that the laptop cluster is continuously
available.

### RAY-FR-007: Safe updates

Updates are initiated by reviewed Git changes. The selected `RayService` upgrade
strategy must account for finite GPU capacity: a blue/green replacement cannot
assume twice the fleet is free. Rollback restores a previously validated image,
configuration, and model revision.

### RAY-FR-008: Observability

The platform exports at least:

- request count, status, queue depth, and cancellation rate;
- time to first token, inter-token latency, total latency, and tokens per second;
- replica health, restart count, scheduling delay, and model load time;
- GPU utilization, VRAM, temperature, power, and throttling indicators;
- node readiness, filesystem pressure, and cache utilization; and
- the model, runtime, prompt/configuration, and hardware class serving each request.

The Ray dashboard and control endpoints remain private. Logs and metrics must not
contain prompts, generated content, credentials, or authorization headers by
default.

### RAY-FR-009: Security

Ray and model pods use dedicated service accounts with least-privilege RBAC,
default-deny network policy, read-only configuration mounts, and no host access
beyond the validated GPU runtime requirement. The service does not receive
production Kubernetes credentials. Secrets use the cluster's approved secret
delivery mechanism and are independently scoped from production credentials.

### RAY-FR-010: Refinery compatibility

The design leaves capacity for Ray Jobs or Kubernetes Jobs to perform bounded model
refinery work. Batch work is lower priority than interactive inference, produces
content-addressed outputs, and cannot evict the gateway, Ray head, or minimum live
model replicas.

### RAY-FR-011: NVIDIA GPU Operator dependency

NVIDIA Ray workers depend on a healthy, GitOps-managed NVIDIA GPU Operator
installation configured for RKE2's containerd socket. The operator supplies the
device plugin, container-toolkit integration, GPU discovery, validation, and DCGM
metrics. Driver installation is either operator-managed or host-managed for each
hardware class, never both.

The Ray worker pod requests `nvidia.com/gpu` and does not become ready until the
device is usable inside the container. Whether the pod specifies
`runtimeClassName: nvidia` is determined by the pinned compatibility matrix: a
validated CDI-based stack may not require it, while an older runtime integration
may. Unsupported or quarantined NVIDIA nodes must not receive operator operands or
Ray GPU workers.

## Non-functional requirements

- **Isolation:** a Beowulf cluster failure cannot impair unrelated production
  workloads.
- **Replaceability:** loss and reprovisioning of a worker cannot destroy the only
  copy of configuration, model artifacts, or evaluation results.
- **Measurability:** placement and promotion decisions use sustained measurements,
  not nominal GPU specifications.
- **Bounded recovery:** readiness is withdrawn promptly and clients receive a
  bounded failure or gateway fallback rather than hanging indefinitely.
- **Reproducibility:** a declared hardware class can be rebuilt from versioned
  provisioning and GitOps inputs.
- **Portability:** the production gateway contract does not expose Ray-specific
  control APIs, allowing another backend to replace or complement Ray Serve.

## Implementation phases

### Phase 0: Compatibility spike

1. Select one qualified NVIDIA laptop and one small, license-compatible model.
2. Select host- or operator-managed driver ownership for its hardware class.
3. Validate RKE2's containerd socket, GPU Operator operands, GPU labels,
   `nvidia.com/gpu` capacity, a pinned CUDA sample, and DCGM metrics.
4. Validate the Ray image and inference engine against that GPU stack.
5. Record cold-load time, VRAM use, sustained throughput, thermals, and failure
   behavior.
6. Pin the first supported version matrix and reject unsupported node classes.

### Phase 1: KubeRay foundation

1. Reconcile the KubeRay operator through GitOps.
2. Deploy a CPU-only `RayService` smoke application.
3. Verify status, logs, metrics, update, rollback, and node-drain behavior.

### Phase 2: Single-GPU model service

1. Add one GPU worker group with the required affinity and laptop toleration.
2. Deploy one model replica with a fixed resource allocation.
3. Exercise an OpenAI-compatible request from inside the Beowulf cluster.
4. Validate cache warmup, readiness, graceful shutdown, and replica restart.

### Phase 3: Production gateway integration

1. Expose one private Beowulf ingress endpoint.
2. Configure a dedicated production gateway backend and model alias.
3. Test authentication, TLS, timeout, circuit breaking, rate limiting, and fallback.
4. Demonstrate that full Beowulf cluster loss leaves other production routes healthy.

### Phase 4: Heterogeneous expansion

1. Add worker groups only for validated hardware classes.
2. Route independent models or replicas to suitable classes.
3. Introduce bounded autoscaling and priority controls.
4. Compare cross-node parallelism with independent serving before adopting it.

### Phase 5: Reliability and refinery

1. Decide whether GCS fault tolerance is justified.
2. Add load, soak, thermal, worker-loss, and head-loss tests.
3. Add a lower-priority refinery workload with reproducible artifacts.
4. Define promotion gates for models and configurations.

## Acceptance tests for the first implementation

- GitOps can create and remove the complete Ray service without manual repair.
- A model pod cannot schedule without both the laptop toleration and approved-class
  affinity.
- GPU Operator validators are healthy and the node exposes the expected discovery
  labels, `nvidia.com/gpu` capacity, and DCGM metrics.
- The selected GPU is visible and exclusively accounted for inside the worker pod.
- A client receives a correct response through the private ingress and production
  gateway path.
- Readiness remains false until model loading completes.
- Killing a replica results in a replacement or a bounded, observable failure.
- Draining the GPU node stops new requests and does not lose authoritative state.
- Removing the Beowulf route does not affect other production gateway backends.
- Metrics identify latency, throughput, errors, GPU pressure, and the serving
  hardware class.

## Proposed repository scaffold

The implementation is expected to introduce a structure similar to:

```text
clusters/beowulf/                 # Flux or equivalent cluster root
infrastructure/kuberay/           # operator, RBAC, policies, monitoring
apps/ray-serve/base/              # shared RayService and ingress resources
apps/ray-serve/models/            # model-specific overlays and configuration
images/ray-serve/                 # reproducible runtime image definitions
tests/smoke/                      # API, placement, drain, and recovery tests
```

This document scopes the layout; directories are created when their first reviewed
implementation artifact is added.

## Open decisions

- Which retained GPU class is the first supported target?
- Does the first engine use Ray Serve LLM with vLLM, SGLang, or a plain Ray Serve
  deployment around another compatible runtime?
- Where will authoritative model artifacts and evaluation results live?
- Which ingress and private-network mechanism connect the production gateway?
- What latency and availability objectives are appropriate for a best-effort local
  backend?
- Is external Redis for GCS fault tolerance justified by measured recovery needs?
- Which upgrade strategy fits the available GPU headroom?

## References

- [Ray Serve production guide](https://docs.ray.io/en/latest/serve/production-guide/)
- [Deploy Ray Serve on Kubernetes](https://docs.ray.io/en/latest/serve/production-guide/kubernetes.html)
- [Ray Serve LLM](https://docs.ray.io/en/latest/serve/llm/)
- [Ray Serve LLM configuration](https://docs.ray.io/en/latest/serve/llm/user-guides/configuration.html)
- [Ray Serve fault tolerance](https://docs.ray.io/en/latest/serve/production-guide/fault-tolerance.html)
- [KubeRay documentation](https://ray-project.github.io/kuberay/)
- [RKE2 GPU Operators](https://docs.rke2.io/add-ons/gpu_operators)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
