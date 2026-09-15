---
name: gke-node-selector-troubleshooting
description: >-
  Diagnoses GKE pods stuck in Pending due to nodeSelector, nodeAffinity, or topologySpreadConstraints failing to match any active node labels. Use when pod events display 0/N nodes available because node selector or required node affinity did not match. Don't use for node taint rejections (use gke-taint-toleration-troubleshooting).
---

# GKE Node Selector Mismatch Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload placement failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.NODES_DIDNT_MATCH_SELECTOR / 0/N nodes match node selector.

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
Observed Symptom: Pod stuck in Pending. Events report: 0/N nodes are available:
N node(s) didn't match Pod's node affinity/selector.

Execute read-only diagnostic commands:
```bash
# 1. Extract pod nodeSelector and nodeAffinity requirements
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{"nodeSelector: "}{.spec.nodeSelector}{"\nnodeAffinity: "}{.spec.affinity.nodeAffinity}{"\n"}'

# 2. Extract pod topologySpreadConstraints
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{.spec.topologySpreadConstraints}'

# 3. Query all active node pool labels across the cluster
kubectl get nodes --show-labels

# 4. Check specific target label keys on all nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,LABELS:.metadata.labels
```

---

### Step 2: Diagnostic Decision Tree
- Typographical Error in Label Key/Value: nodeSelector specifies a key or
  value that does not exist on any node (e.g. cloud.google.com/gke-nodepool:
  backend-pool when pool is named backend).
- Node Pool Deleted or Scaled to 0: The targeted node pool exists in GKE
  configuration but currently has zero running nodes and autoscaler cannot
  provision it.
- Topology Spread Constraint Unsatisfiable: topologySpreadConstraints with
  whenUnsatisfiable: DoNotSchedule cannot find a node in an under-represented
  zone without violating maxSkew.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If label typo: Update the pod's nodeSelector or nodeAffinity in the
   Deployment YAML to match actual node pool labels.
2. If targeted node pool empty: Scale the targeted node pool or check
   autoscaler configuration.
3. For topology constraints: Change whenUnsatisfiable: DoNotSchedule to
   ScheduleAnyway if strict regional distribution is not mandatory.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
