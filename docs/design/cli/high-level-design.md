# Karta CLI - Product Design

## Background

Karta is a CRD-based Go library that provides a universal abstraction for any Kubernetes workload type. Using JQ-based Karta definitions, it can extract structure (components, hierarchy), status (phase, conditions), scaling (replicas, min/max), and pod specs from any CRD - PyTorchJob, RayCluster, JobSet, KServe, and others.

We believe Karta's abstraction layer can serve as the foundation for a tool that brings visibility to the k8s workload space - a CLI/web/MCP that understands any workload type out of the box for a live cluster.

### What exists today for workload visibility

- [**kubectl**](https://kubernetes.io/docs/reference/kubectl/) - works per resource type. `kubectl get pytorchjob` shows the CRD, `kubectl get pods` shows a flat list. No connection between them. Each CRD type requires different knowledge to inspect.
- [**k9s**](https://k9scli.io/) / [**Lens**](https://k8slens.dev/) - general-purpose cluster browsers. They show resources by kind but have no concept of a "workload" as a composed unit of multiple resource types.
- [**kubectl-tree**](https://github.com/ahmetb/kubectl-tree) - shows the ownership chain (Pod → ReplicaSet → Deployment) but has no semantic understanding. It can't tell you which pod is a master and which is a worker. No phase or status parsing - it doesn't understand the workload's lifecycle state. No scaling awareness - it shows individual objects but doesn't understand replicas, min/max, or autoscaling.

### What's missing

No existing tool provides **workload-aware** visibility across different CRD types. The questions that keep coming up and we think that Karta can answer:

- What workloads are running in my cluster, across all types, and what's their status?
- What does the structure of a workload look like - which pods play which role (master, worker, head)?
- How many GPUs does a workload consume across all its components?

Every workload type (PyTorchJob, RayCluster, JobSet, KServe) structures this information differently. Today, answering these questions requires per-CRD knowledge and manual inspection.

Karta already has the abstraction layer to read any workload type uniformly. A CLI (and later web/MCP) can build on this to provide the missing visibility.

---

## Karta definition resolution

We want the CLI to be useful immediately - without requiring users to install Karta CRDs into their cluster first. If someone has PyTorchJobs running, `karta workload list` should find them out of the box.

To make this work, the CLI would ship with **community definitions** - the Karta definitions from `docs/examples/` embedded in the binary. These are maintained by the open-source community, cover standard frameworks (PyTorchJob, RayCluster, JobSet, KServe, etc.), and are tested against each framework's stable API. They update automatically when you upgrade the CLI.

For users who need more - custom CRDs, org-specific status mappings, different pod selectors - they can apply Karta definitions to the cluster as CRDs. These **cluster definitions** take precedence over community ones when both target the same workload type.

The `ORIGIN` column in `karta definition list` shows where each definition came from (`community` or `cluster`).

---

## High-level design

### Command structure

Two top-level nouns:

```shell
karta workload ...        # operational: what's running in my cluster
karta definition ...      # meta: what workload types does Karta understand
```

### `karta workload list`

List all workloads discovered via all known Kartas (community + cluster).

```shell
$ karta workload list -n ml-team
NAMESPACE  NAME              KIND         PHASE       COMPONENTS               AGE
ml-team    llama-finetune    PyTorchJob   Running     master(1), worker(4)     2h
ml-team    embed-svc         KServe       Running     predictor(2)             5d
ml-team    preprocess        JobSet       Completed   etl(3)                   1h
ml-team    ray-train         RayCluster   Degraded    head(1), gpu-worker(3)   30m
ml-team    my-inference      CustomJob    Running     runner(2)                4h
```

### `karta workload tree <name>`

Hierarchical tree view: workload -> components -> instances -> pods with status and resources.

Simple workload (PyTorchJob):

```shell
$ karta workload tree llama-finetune
PyTorchJob/llama-finetune [Running]
├── master (1 replica)
│   └── Pod/llama-finetune-master-0    Running   gpu: 1   node-01
└── worker (4 replicas)
    ├── Pod/llama-finetune-worker-0    Running   gpu: 8   node-02
    ├── Pod/llama-finetune-worker-1    Running   gpu: 8   node-03
    ├── Pod/llama-finetune-worker-2    Running   gpu: 8   node-04
    └── Pod/llama-finetune-worker-3    Pending   gpu: 8   <none>
```

Complex workload (Dynamo - multi-instance with nested children):

```shell
$ karta workload tree my-dynamo-graph
DynamoGraphDeployment/my-dynamo-graph [Running]
└── service
    ├── Frontend (1 replica)
    │   ├── Pod/frontend-0    Running   gpu: 1   node-01
    │   └── Pod/frontend-1    Running   gpu: 1   node-02
    ├── PrefillWorker (2 replicas)
    │   ├── Pod/prefill-0     Running   gpu: 8   node-03
    │   ├── Pod/prefill-1     Running   gpu: 8   node-04
    │   ├── Pod/prefill-2     Running   gpu: 8   node-05
    │   └── Pod/prefill-3     Running   gpu: 8   node-06
    └── DecodeWorker (4 replicas)
        ├── Pod/decode-0      Running   gpu: 4   node-07
        ├── Pod/decode-1      Running   gpu: 4   node-08
        ├── Pod/decode-2      Running   gpu: 4   node-09
        └── Pod/decode-3      Running   gpu: 4   node-10
```

### `karta workload status <name>`

Workload status as Karta sees it - normalized phases and per-component readiness.

```shell
$ karta workload status llama-finetune
Workload:  PyTorchJob/llama-finetune
Phase:     Running
Age:       2h15m

Components:
  master:  1/1 pods running
  worker:  3/4 pods running
```

### `karta workload resources <name>`

Resource breakdown for a single workload by component.

```shell
$ karta workload resources llama-finetune
COMPONENT   REPLICAS   CPU(req)   MEM(req)   GPU
master      1          4          16Gi       1
worker      4          16         64Gi       32
────────────────────────────────────────────────
TOTAL       5          20         80Gi       33
```

### `karta definition list` / `karta definition describe <name>` / `karta definition validate <file>`

Inspect the Karta definitions the CLI knows about - both community and cluster-provided.

```shell
$ karta definition list
NAME                              KIND               ORIGIN     COMPONENTS
kubeflow-org-pytorchjob-v1        PyTorchJob         community    pytorchjob, master, worker
ray-io-raycluster-v1              RayCluster         community    raycluster, head, worker
my-custom-workload                CustomJob          cluster    customjob, runner

$ karta definition describe kubeflow-org-pytorchjob-v1
(shows structure tree, status mappings, gang scheduling config)

$ karta definition validate ./my-custom-karta.yaml
✓ Valid Karta definition
```

---

## Data Model

The CLI, web dashboard, and MCP all need the same underlying data - a workload's component hierarchy with pods, scale, status, and resources. Rather than each consumer assembling this independently, the Karta library should produce a single `WorkloadTree` that any consumer can traverse and render in its own way.

Beyond the visibility tool, this tree can serve as a shared language between components. The Run:ai External Workload Integrator (EWI) already builds a similar tree for its complex workload structure feature (tracking hierarchies like Dynamo with multi-instance components and nested children) - it could adopt `WorkloadTree` as its foundation. A user submitting a workload could use the same tree structure to set desired topology or other instructions pre-submission.

The model is split into two layers: `WorkloadTree` is the raw data produced by the Karta library (structure, scale, specs, matched pods - no computation). `WorkloadView` is the display layer built on top of it by the CLI (resource aggregation, ready counts, pod details).

### `WorkloadTree` - shared utility (`pkg/tree/`)

A generic tree built from a Karta definition + workload object + matched pods. Contains the raw Karta-extracted data - structure, scale, extracted specs, status, and which pods belong where. This lives in `pkg/` so any consumer can use it.

The structure was designed with the Run:ai External Workload Integrator (EWI) as a reference consumer - recently a feature was added to the EWI that builds complex hierarchical trees for workloads like Dynamo (multi-instance components with nested children per instance). The `WorkloadTree` is shaped so the EWI could adopt it as its tree-building foundation and add its own persistence layer on top.

```go
// WorkloadTree is the raw tree produced by the shared builder.
type WorkloadTree struct {
    Status     WorkloadStatus              // from root component GetStatus()
    Children   []ComponentNode             // root-level components
}

type WorkloadStatus struct {
    Phases []string                        // matched phases, can be multiple (e.g. ["Running", "Degraded"])
}

type ComponentNode struct {
    Name       string
    Kind       *GroupVersionKind            // nil for logical grouping components
    Instances  []InstanceNode              // always at least one (see below)
}

// InstanceNode represents one instance of a component.
// Every component has at least one instance. When InstanceKey is nil,
// it means the component is not multi-instance - there's just one.
// For multi-instance components (e.g., Dynamo services with "Frontend",
// "PrefillWorker"), each instance has its own InstanceKey and potentially
// its own child components underneath.
type InstanceNode struct {
    InstanceKey       *string              // nil = single instance (not multi-instance)
    ReplicaKey        *string              // nil = not replicated
    Scale             *Scale               // from Component.GetScale()
    ExtractedInstance *ExtractedInstance    // from Component.GetExtractedInstances() - pod specs, metadata
    Pods              []*corev1.Pod         // live pods matched to this instance
    Children          []ComponentNode       // child components under this instance
}

type Scale struct {
    Replicas    *int32
    MinReplicas *int32
    MaxReplicas *int32
}
```

### Tree builder (`pkg/tree/builder.go`)

```go
type PodMatcher interface {
    Matches(ctx context.Context, pod *corev1.Pod, node *ComponentNode) (bool, error)
}

func Build(ctx context.Context, ri *ResourceInterface, workload client.Object, pods []corev1.Pod, matcher PodMatcher) (*WorkloadTree, error)
```

The `PodMatcher` decouples the tree structure from the matching strategy. Each consumer provides its own matching logic - the builder just asks "does this pod belong here?" and places it accordingly.

The builder works top-down, component by component. At each component, it uses the matcher to find which pods belong to it or its descendants, then distributes them across instances. Each instance passes only its pods down to child components, narrowing the set at each level until pods reach their leaf component.

Example for a PyTorchJob with 5 pods (master-0, worker-0..3):

```shell
[5 pods] → master component: matcher claims master-0 → [4 remaining]
         → worker component: matcher claims worker-0..3
```

Example for Dynamo with 6 pods (2 frontend, 4 prefill):

```shell
[6 pods] → service component: matcher claims all 6
           → "Frontend" instance: 2 pods
             → lws child: matcher claims them → placed
           → "PrefillWorker" instance: 4 pods
             → lws child: matcher claims them → placed
```

### `WorkloadView` - CLI/web layer

The CLI walks the `WorkloadTree` and computes display data as it traverses: resource aggregation (CPU, memory, GPU summed up the tree), ready counts ("3/4"), and pod details (phase, node). This `WorkloadView` is what gets rendered as table, tree, JSON, or served via web/MCP.

```go
type WorkloadView struct {
    Name       string
    Namespace  string
    Kind       string              // "PyTorchJob", "RayCluster", etc.
    APIVersion string
    Age        time.Duration
    CreatedAt  time.Time

    Phases     []string            // from WorkloadTree.Status.Phases
    Resources  ResourceSummary     // aggregated across all components (computed)
    Children   []ComponentView
}

type ComponentView struct {
    Name       string
    Kind       string
    Instances  []InstanceView
    Scale      ScaleView
    Resources  ResourceSummary     // aggregated across instances (computed)
    ReadyCount string              // "3/4" (computed from live pods)
}

type ScaleView struct {
    Replicas    int32
    MinReplicas *int32
    MaxReplicas *int32
}

type InstanceView struct {
    InstanceKey *string             // nil = single instance
    ReplicaKey  *string
    Resources   ResourceSummary    // from extracted pod spec (computed)
    Pods        []PodView
    Children    []ComponentView    // child components under this instance
}

type PodView struct {
    Name       string
    Phase      string              // Running, Pending, Succeeded, Failed
    Ready      bool
    Node       string
    Resources  ResourceSummary     // from live pod spec (computed)
}

type ResourceSummary struct {
    CPUMillis    int64
    MemoryBytes  int64
    GPUs         int64
    Extended     map[string]int64
}
```

