---
name: gke-scheduling-and-placement
metadata:
  category: Containers
description: >-
  Diagnoses complex GKE Kubernetes scheduling rejections caused by nodeSelector, nodeAffinity, podAntiAffinity, topologySpreadConstraints, and node taints/tolerations. Use when pods remain stuck in Pending with "didn't match node selector", "had taints that the pod didn't tolerate", or "0/N nodes available". Don't use for cluster autoscaler provisioning limits (use gke-cluster-autoscaler).
---

# GKE Scheduling & Placement Constraint Troubleshooting Skill

Use this skill to diagnose why a Kubernetes Pod cannot be scheduled on any node in a GKE cluster due to placement constraints, label mismatches, or untolerated taints. When simple resource checks (CPU and memory requests) show sufficient allocatable capacity, scheduling stalls are typically caused by multi-dimensional constraint mismatches between the pod's placement specification and the cluster's node pool topologies.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: evaluate constraints first, then output a corrected pod manifest or node pool command.

## 🔍 Diagnostic Workflow

### Step 0: Context Discovery

1. **Parameter Discovery**: Extract `project_id`, `cluster_name`, `cluster_location`, `pod_name`, and `namespace` non-interactively.
2. **Retrieve Scheduling Error Event**:
   ```bash
   kubectl get events -n "{namespace}" \
     --field-selector="involvedObject.name={pod_name},reason=FailedScheduling" \
     --sort-by='.metadata.creationTimestamp' \
     --output=custom-columns='TIME:.metadata.creationTimestamp,MESSAGE:.message'
   ```

--------------------------------------------------------------------------------

### Step 1: Extract Pod Placement Constraints

Inspect all scheduling criteria declared in the Pod specification:

```bash
# Extract nodeSelector, nodeAffinity, podAntiAffinity, tolerations, and topologySpreadConstraints
kubectl get pod "{pod_name}" -n "{namespace}" -o jsonpath='{
  "nodeSelector": {.spec.nodeSelector},
  "nodeAffinity": {.spec.affinity.nodeAffinity},
  "podAntiAffinity": {.spec.affinity.podAntiAffinity},
  "tolerations": {.spec.tolerations},
  "topologySpread": {.spec.topologySpreadConstraints}
}'
```

--------------------------------------------------------------------------------

### Step 2: Query Active Node Pool Topologies and Taints

Fetch the labels and taints across all nodes in the cluster to identify which constraint eliminates each node pool:

```bash
# 1. List node pools and their machine types
gcloud container node-pools list --cluster="{cluster_name}" --location="{cluster_location}" --project="{project_id}"

# 2. Query node labels and taints across all nodes
kubectl get nodes -o custom-columns='
  NAME:.metadata.name,
  NODEPOOL:.metadata.labels.cloud\.google\.com/gke-nodepool,
  ZONE:.metadata.labels.topology\.kubernetes\.io/zone,
  TAINTS:.spec.taints[*].key
'
```

--------------------------------------------------------------------------------

### Step 3: Constraint Elimination Solver

Evaluate each node pool against the pod's constraints:

1. **Node Selector Evaluation**:
   - Check whether any active node possesses the exact key-value pairs specified in `spec.nodeSelector`.
   - *Common error*: Typo in zone or tier label (e.g. `topology.kubernetes.io/zone: us-central1-a` on a cluster running in `asia-southeast1`).
2. **Taint and Toleration Evaluation**:
   - For every node reporting a taint (`key:value:effect`), verify whether the pod declares a matching toleration (`key`, `operator: Equal` or `Exists`, `effect`).
   - *Common error*: Scheduling general workloads onto dedicated tenant pools or GPU/TPU accelerator pools without tolerations.
3. **Topology Spread Constraints**:
   - Check if `maxSkew` is violated across failure domains (e.g. zones).

--------------------------------------------------------------------------------

### Step 4: Corrective Manifest Generation

Output the exact GitOps YAML patch:

#### Example 4a: Add Missing Toleration to Pod Spec:
```yaml
spec:
  tolerations:
  - key: "cloud.google.com/gke-nodepool"
    operator: "Equal"
    value: "{target_nodepool}"
    effect: "NoSchedule"
```

#### Example 4b: Correct Invalid Node Selector:
```yaml
spec:
  nodeSelector:
    topology.kubernetes.io/zone: "{valid_cluster_zone}"
```
