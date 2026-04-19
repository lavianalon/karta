# Karta CLI - Implementation Details

See [high-level-design.md](high-level-design.md) for the product context, feature set, and data model.

---

## Build Algorithm Detail

The tree builder (`pkg/tree/builder.go`) constructs a `WorkloadTree` recursively:

```
Build(ri, workload, pods, matcher):

  1. EXTRACT from workload object (using existing Karta library)
     factory = NewComponentFactoryFromObject(ri, workload)
     status  = factory.GetRootComponent().GetStatus() -> phases
     For each component: extract instances, scale, specs

  2. BUILD TREE recursively, starting with root-level child components

     buildComponents(componentDefs, availablePods):

       for each componentDef (topologically sorted):
         node = ComponentNode{Name, Kind}

         // Ask matcher: which pods belong to this node or its descendants?
         matched, remaining = partition(availablePods, pod =>
           matcher.Matches(pod, &node)
         )

         // Distribute matched pods across instances
         for each instance extracted from workload spec:
           instanceNode = InstanceNode{InstanceKey, Scale, ExtractedInstance}
           instanceNode.Pods = matched pods whose instance key matches

           // Recurse: child components under this instance
           // only see pods that were placed in this instance
           instanceNode.Children = buildComponents(
             childDefsOf(componentDef),
             instanceNode.Pods,
           )

         node.Instances = append(instanceNode...)

       return nodes

  3. RETURN WorkloadTree{Status, Children}
```

The builder works top-down, component by component. At each component, it uses the matcher to find which pods belong to it or its descendants, then distributes them across instances. Each instance passes only its pods down to child components, narrowing the set at each level until pods reach their leaf component.

---

## WorkloadView Types

The CLI view model - built by walking a `WorkloadTree` and computing display data.

```go
type WorkloadView struct {
    Name       string
    Namespace  string
    Kind       string              // "PyTorchJob", "RayCluster", etc.
    APIVersion string              // "kubeflow.org/v1", etc.
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
    Nodes      []string            // distinct nodes hosting this component's pods (computed)
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

### Resource extraction across the 3 spec patterns

The Karta library supports three mutually exclusive ways to define pod specs. The enricher normalizes all three into `ResourceSummary`:

| Spec pattern | How resources are extracted |
|---|---|
| **PodTemplateSpec** | `podTemplateSpec.Spec.Containers[].Resources.Requests` |
| **PodSpec + Metadata** | `podSpec.Containers[].Resources.Requests` |
| **FragmentedPodSpec** | `fragmentedPodSpec.Resources`, OR `fragmentedPodSpec.Containers[].Resources.Requests`, OR `fragmentedPodSpec.Container.Resources.Requests` |

GPU count = sum of `nvidia.com/gpu` (and `amd.com/gpu`, etc.) from requests across all containers.

---

## Technical Architecture

### Project Structure
```
karta/
├── cmd/karta/
│   └── main.go                    # Cobra root, global flags (kubeconfig, context, namespace)
├── pkg/                           # Karta library
│   ├── api/optimization/v1alpha1/ # CRD types
│   ├── resource/                  # ComponentFactory, Component, PodQuerier, Accessor
│   ├── instructions/              # StructureSummary, InferPodComponent, gang scheduling
│   ├── jq/                        # JQ execution engine
│   └── tree/                      # NEW: shared WorkloadTree builder
│       ├── types.go               # WorkloadTree, ComponentNode, InstanceNode, PodMatcher
│       └── builder.go             # Build() - tree construction algorithm
├── internal/
│   ├── cli/                       # Command implementations
│   │   ├── root.go                # Root command + global flags
│   │   ├── workload_list.go       # karta workload list
│   │   ├── workload_tree.go       # karta workload tree
│   │   ├── workload_status.go     # karta workload status
│   │   ├── workload_resources.go  # karta workload resources
│   │   └── definition.go          # karta definition {list,describe,validate}
│   ├── k8s/                       # Kubernetes client layer
│   │   ├── client.go              # Client init from kubeconfig
│   │   ├── discovery.go           # Resolve Kartas (community + cluster) + discover workloads
│   │   └── pods.go                # List pods in namespace
│   ├── community/                 # Embedded Karta definitions
│   │   └── definitions.go         # //go:embed docs/examples/*.yaml + loader
│   ├── matcher/                   # CLI's PodMatcher implementation
│   │   └── matcher.go             # Hybrid: ComponentTypeSelector + owner-ref walking + logical grouping
│   ├── view/                      # WorkloadView - display model built from WorkloadTree
│   │   ├── types.go               # WorkloadView, ComponentView, PodView, ResourceSummary
│   │   └── enricher.go            # Walks WorkloadTree, computes resources + ready counts
│   └── output/                    # Rendering
│       ├── table.go               # Table formatter (text/tabwriter)
│       ├── tree.go                # ASCII tree renderer
│       └── json.go                # JSON/YAML output
├── docs/examples/                 # 11 community Karta definition YAMLs (embedded in CLI)
└── charts/                        # Helm chart
```

### Data Flow

```
internal/community/     -->  Load 11 embedded Karta definitions
    │
    ├── merge <-- Kubernetes API (list cluster Kartas if CRD exists)
    │
    ▼
internal/k8s/           -->  For each effective Karta, list workloads matching root GVK
    │                         Skip GVKs not present in cluster (no CRD = no workloads)
    │                         List pods in namespace
    ▼
pkg/tree/               -->  Build(ri, workload, pods, matcher) -> WorkloadTree
(shared builder)              Topological sort, pod matching, extraction
    │
    ▼
internal/view/          -->  Walk WorkloadTree -> compute resources, ready counts -> WorkloadView
    │
    ├──> internal/output/   -->  CLI: table / tree / wide
    ├──> net/http handler   -->  Web: JSON API
    └──> MCP tool handler   -->  MCP: JSON response
```

### Key Integration Points (existing code to reuse)

| File | What it provides | Used by |
|---|---|---|
| `pkg/resource/component_factory.go` | `NewComponentFactoryFromObject(ri, obj)` - entry point | `pkg/tree/builder.go` |
| `pkg/resource/component.go` | `GetExtractedInstances()`, `GetScale()`, `GetStatus()` | `pkg/tree/builder.go` |
| `pkg/resource/pod_querier.go` | `MatchesComponentType()`, `GetMatchingInstanceId()` | `internal/matcher/matcher.go` |
| `pkg/instructions/pod.go` | `InferPodComponent()`, `InferPodComponentInstance()` | `internal/matcher/matcher.go` |
| `pkg/instructions/summary.go` | `NewStructureSummary(ri)` | `internal/matcher/matcher.go` |
| `pkg/api/optimization/v1alpha1/types.go` | Karta CRD types | discovery, builder |
| `pkg/api/optimization/v1alpha1/validation.go` | Karta validation | `karta definition validate` |

### Karta Definition Resolution (`internal/k8s/discovery.go`)

```
┌──────────────────────────────┐    ┌──────────────────────────────┐
│  Community Kartas (embedded) │    │  Cluster Kartas (if CRD      │
│  11 definitions from         │    │  is installed, list them)     │
│  docs/examples/*.yaml        │    │  May include custom defs      │
└──────────┬───────────────────┘    └──────────┬───────────────────┘
           │                                   │
           └───────────┬───────────────────────┘
                       ▼
              Merge: cluster wins on GVK conflict
                       │
                       ▼
              Effective Karta set (deduplicated by root GVK)
```

### Workload Discovery Algorithm

1. **Resolve Kartas**: Load community definitions (embedded YAML). If the Karta CRD exists in the cluster, list cluster Kartas too. Merge: cluster version wins when both define the same root GVK.
2. For each Karta in the effective set, extract root component GVK (e.g., `kubeflow.org/v1/PyTorchJob`)
3. Use dynamic client (`k8s.io/client-go/dynamic`) to list all instances of that GVK in target namespace(s). Skip GVKs that don't exist in the cluster (the CRD isn't installed - no workloads of that type).
4. For each workload instance + its pods, call `pkg/tree.Build(ri, workload, pods, matcher)` to get a `WorkloadTree`
5. Walk the tree in `internal/view/enricher.go` to build a `WorkloadView` (compute resources, ready counts)

### New Dependencies
- `github.com/spf13/cobra` - CLI framework (standard for K8s tools)
- `k8s.io/client-go` - already indirect dep, promote to direct for REST config + dynamic client

### Changes to existing `pkg/`
- **New package `pkg/tree/`** - the shared `WorkloadTree` builder, `PodMatcher` interface, and types. This is new code added to the library, not modifications to existing code.

---

## Implementation Phases

### Phase 1: `pkg/tree/` + `karta workload list` (foundation)
1. Implement `pkg/tree/types.go` - `WorkloadTree`, `ComponentNode`, `InstanceNode`, `PodMatcher` interface
2. Implement `pkg/tree/builder.go` - `Build()` with the recursive tree construction algorithm
3. Implement `internal/matcher/matcher.go` - CLI's `PodMatcher` using all 3 strategies (selector, owner-ref walking, logical grouping)
4. Create `cmd/karta/main.go` with Cobra root command + `workload` and `definition` subcommands
5. Add global flags: `--kubeconfig`, `--context`, `-n/--namespace`, `-A`, `-o`
6. Implement `internal/community/definitions.go` - embed `docs/examples/*.yaml`, parse into Karta objects
7. Implement `internal/k8s/client.go` - K8s client from kubeconfig
8. Implement `internal/k8s/discovery.go` - Karta resolution (community + cluster merge) + workload discovery via dynamic client
9. Implement `karta workload list` with table output
10. Implement `karta definition list` / `karta definition describe` / `karta definition validate`

### Phase 2: `karta workload tree` + `karta workload status` (highest user value)
1. Implement `internal/view/types.go` - `WorkloadView`, `ComponentView`, `InstanceView`, `PodView`, `ResourceSummary`
2. Implement `internal/view/enricher.go` - walk `WorkloadTree`, compute resources, ready counts, and node distribution per component
3. Implement `internal/output/tree.go` - ASCII tree renderer
4. Implement `karta workload tree` command
5. Implement `karta workload status` command

### Phase 3: `karta workload resources` + polish
1. Implement `karta workload resources` (single workload resource breakdown by component)
2. Add JSON/YAML output for all commands
3. Add `-o wide` format
4. Shell completions, error handling, edge cases

### Phase 4 (future): Web dashboard
- `karta serve --port 8080`
- REST API backed by same view layer
- Lightweight embedded SPA (Go `embed` package)
- Dashboard: workload list, click-through to tree view, resource charts

---

## Verification

1. **Unit tests**:
   - `pkg/tree/builder_test.go` - test tree construction with fixture Karta definitions + mock pods
   - `internal/matcher/matcher_test.go` - test pod matching (selector, owner-ref, logical)
   - `internal/view/enricher_test.go` - test resource aggregation and ready count computation
2. **Integration test**: Use envtest or kind cluster with Karta CRD + sample workloads
3. **Manual testing**:
   - `karta workload list -A` on a cluster with workloads - should work with zero setup (community definitions)
   - Test with cluster Kartas installed too - verify cluster version takes precedence
   - Create sample workloads (at minimum a Deployment-based one, ideally a PyTorchJob)
   - Run each command and verify output matches expected format
   - Test `-o json`, `-o yaml`, `-A`, `-n`, `--type`, `--phase` flags
4. **Build**: `go build ./cmd/karta` produces a single binary

---

## EWI Adoption of `pkg/tree/`

The Run:ai External Workload Integrator (EWI) currently has its own tree-building logic in `structure/algorithms.go`. Once `pkg/tree/` exists, the EWI could adopt it by implementing a custom `PodMatcher` and wrapping `Build()` with its caching layer.

### EWI PodMatcher

The EWI's matcher combines all three strategies, with a fast path for previously labeled pods:

```go
type EWIMatcher struct {
    defsByName  map[string]ComponentDefinition  // from the RI
    ownerChains map[string]OwnerChain           // pre-fetched per pod
}

func (m *EWIMatcher) Matches(ctx context.Context, pod *corev1.Pod, node *ComponentNode) (bool, error) {
    // Fast path: pod already labeled for this component from a previous calculation
    if cachedName := pod.Labels["run.ai/element-..."]; cachedName == node.Name {
        return true, nil
    }

    def := m.defsByName[node.Name]

    hasSelector := def.PodSelector != nil && def.PodSelector.ComponentTypeSelector != nil
    hasKind := def.Kind != nil

    if hasSelector {
        // Strategy A: label/annotation match
        querier := NewPodQuerier(pod)
        return querier.MatchesComponentType(ctx, def.PodSelector.ComponentTypeSelector)
    }

    if hasKind {
        // Strategy B: find Kind in pod's owner chain
        chain := m.ownerChains[pod.Name]
        return findKindInChain(def.Kind, chain) != nil, nil
    }

    // Strategy C: logical grouping, always matches
    return true, nil
}
```

### EWI usage of Build()

```go
// Pre-fetch owner chains (EWI already does this)
ownerChains := fetchOwnerChains(ctx, pods)

// Build tree using shared builder
matcher := &EWIMatcher{defsByName: defsFromRI(ri), ownerChains: ownerChains}
tree, err := tree.Build(ctx, ri, workload, pods, matcher)

// Map WorkloadTree -> StructureElements (EWI-specific)
elements := mapToStructureElements(tree)

// Assign/reuse stable UUIDs by element name
stabilizeIDs(elements, existingElementsByID)

// Label pods for fast path on next recalculation
labelPods(pods, elements)
```

### What changes in the EWI

- `structure/algorithms.go` shrinks significantly - the topological sort, pod partitioning, and tree traversal move into `pkg/tree/builder.go`
- The EWI keeps: owner chain fetching, `EWIMatcher`, `StructureElement` mapping, UUID stability, pod labeling, `IsStructureUnchanged()` fast path
- The `IsStructureUnchanged()` check wraps around `Build()` - if unchanged, skip the call entirely
