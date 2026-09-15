---
name: gke-node-unavailability-troubleshooting
description: >-
  Diagnoses GKE nodes abruptly transitioning to NotReady, NodeStatusUnknown, or terminating unexpectedly. Use when active cluster nodes become unavailable, fail heartbeat lease renewals, or experience kernel/systemd panics. Don't use for initial node join bootstrap failures (use gke-node-notready-troubleshooting).
---

# GKE Node Unavailability Troubleshooting Skill

## Purpose & Scope
This skill diagnoses runtime node health transitions and sudden node unavailability
in Google Kubernetes Engine (GKE) clusters, addressing incidents triggered by
`IssueCategory.NODE_UNAVAILABILITY` or `diagnostics.gke_node_unavailability`.

This skill operates non-interactively and enforces a read-only diagnostics
boundary: inspect node conditions, node lease renewal status, Compute Engine VM
lifecycle states, and underlying system daemon health before proposing recovery
actions for human review.

---

## Diagnostic Workflow

### Step 0: Non-Interactive Context Discovery & Time Window
1. Context Extraction: Extract `project_id`, `cluster_name`, `cluster_location`,
   `node_name`, and `node_pool_name` from the prompt or environment.
2. Time Window: Center a 1-hour query window around the unavailability event
   (`start = event_time - 30m`, `end = event_time + 30m`).

---

### Step 1: Physical Symptom & Event Inspection
Observed Symptom: Node transitions from `Ready` to `NotReady` or `Unknown`. Pods
on the node enter `Terminating` or are rescheduled after the node eviction timeout
(default 5 minutes).

Execute read-only diagnostic commands:
```bash
# 1. Inspect node condition status and last transition timestamps
kubectl get node {node_name} -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'

# 2. Inspect node heartbeat lease in kube-node-lease
kubectl get lease {node_name} -n kube-node-lease -o yaml

# 3. Query node-level lifecycle events
kubectl get events --all-namespaces \
  --field-selector involvedObject.name={node_name},involvedObject.kind=Node \
  --sort-by='.metadata.creationTimestamp'

# 4. Check underlying Compute Engine VM instance status and maintenance events
gcloud compute instances describe {node_name} \
  --zone={zone} --project={project_id} \
  --format="yaml(status,scheduling,lastStartTimestamp,lastStopTimestamp)"

# 5. Inspect host maintenance and preemption events
gcloud compute operations list \
  --project={project_id} \
  --filter="targetLink ~ '{node_name}'" \
  --format="table(name,operationType,status,insertTime)"
```

---

### Step 2: Diagnostic Decision Tree
- **Node Heartbeat Lease Expired (`NodeStatusUnknown`)**: The node controller has
  not received a heartbeat update for >40 seconds. Kubelet stopped posting status
  due to network partition, VM freeze, or kubelet crash.
- **Compute Engine Host Maintenance / Live Migration**: VM instance underwent live
  migration or encountered an unexpected host hardware error.
- **Spot / Preemptible VM Termination**: If using Spot node pools, the underlying
  Compute Engine instance was reclaimed by Google Cloud.
- **Kernel Hang or Memory Pressure (OOM Kill on System Daemons)**: Kernel panic or
  out-of-memory killer terminated `containerd`, `kubelet`, or critical system daemons.
- **Disk Pressure / Docker Root Full**: Root filesystem or container storage disk
  reached 100% capacity, causing kubelet to reject operations and transition to `NotReady`.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. **Spot Preemption**: Verify node pool auto-repair triggers GCE instance
   recreation; recommend mixing on-demand nodes for critical workloads.
2. **Kubelet Crash / System Hang**: Execute node drain/cordon and initiate GKE
   node repair or recreation:
   ```bash
   kubectl cordon {node_name}
   gcloud container clusters reset-nodes {cluster_name} --zone={zone} --node-pool={node_pool}
   ```
3. **Persistent Hardware Failure**: Trigger node auto-repair or manually delete the
   failed GCE instance from the instance group to force clean recreation.

### Operational Guarantees
- **Read-Only Verification**: All diagnostic queries use read-only inspection commands.
- **No Unscheduled Drain**: Never cordon or drain production nodes autonomously.
