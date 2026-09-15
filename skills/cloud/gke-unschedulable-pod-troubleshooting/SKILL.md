---
name: gke-unschedulable-pod-troubleshooting
description: >-
  Diagnoses GKE pods stuck in Pending due to insufficient compute capacity (insufficient CPU, memory, or ephemeral storage) across all active nodes. Use when pod events display 0/N nodes available due to resource capacity constraints. Don't use for node label constraint mismatches (use gke-node-selector-troubleshooting) or node taints (use gke-taint-toleration-troubleshooting).
---

# GKE Unschedulable Pod Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload placement failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueDetail.Type.POD_UNSCHEDULABLE / Insufficient cpu, memory, ephemeral-
storage.

This skill operates non-interactively and enforces a read-only diagnostics
boundary: gather evidence first, correlate failure signatures, and propose
GitOps manifests or administrative actions for human review before any change
reaches production.

---

## Diagnostic Workflow

### Step 0: Non-Interactive Context Discovery & Time Window
1. Context Extraction: Extract project_id, cluster_name, cluster_location,
   namespace, and target resource names from the prompt or environment.
2. Time Window: Center a 1-hour query window around the incident
   (start = issue_time - 30m, end = issue_time + 30m).

---

### Step 1: Physical Symptom & Event Inspection
Observed Symptom: Pod remains in Pending phase. Events report: 0/N nodes are
available: N Insufficient cpu, N Insufficient memory.

Execute read-only diagnostic commands:
```bash
# 1. Inspect pod scheduling failure events
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pod_name} \
  --sort-by='.metadata.creationTimestamp'

# 2. Inspect pod resource requests
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .spec.containers[*]}{.name}{" CPU Req: "}{.resources.requests.cpu}{" Mem Req: "}{.resources.requests.memory}{"\n"}{end}'

# 3. Inspect node allocatable capacity and existing resource allocations
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,CPU_ALLOC:.status.allocatable.cpu,MEM_ALLOC:.status.allocatable.memory,PODS:.status.allocatable.pods

# 4. Check cluster autoscaler status and scale-up decisions
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
```

---

### Step 2: Diagnostic Decision Tree
- Insufficient CPU / Memory on Fixed Pool: Active node pool has reached max
  size or autoscaling is disabled, and existing nodes are fully booked.
- Single Request Exceeds Node Capacity: Pod requested more CPU or memory than
  any single node type in the cluster provides (e.g., requesting 32 CPUs on
  e2-standard-16 nodes).
- Autoscaler Blocked: Cluster autoscaler reached maxNodes limit on node pool
  or hit Compute Engine quota. Proceed to gke-autoscaler-troubleshooting or gcp-
  compute-quota-troubleshooting.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If resource requests oversized: Right-size container resources.requests
   based on actual historical utilization.
2. If cluster full: Increase the node pool maximum node count or enable
   cluster autoscaling.
3. If node type too small: Add a new node pool with larger machine types
   capable of hosting the workload.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
