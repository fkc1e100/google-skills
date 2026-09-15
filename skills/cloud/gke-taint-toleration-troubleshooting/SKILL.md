---
name: gke-taint-toleration-troubleshooting
description: >-
  Diagnoses GKE pods stuck in Pending due to node taints (NoSchedule, NoExecute) that lack corresponding tolerations on the pod specification. Use when pod events display node(s) had untolerated taint or workloads fail to schedule on specialized accelerator or tenant isolation nodes. Don't use for label selector mismatches (use gke-node-selector-troubleshooting).
---

# GKE Node Taint & Toleration Mismatch Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload placement failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.NODES_TAINTS_POD_DIDNT_TOLERATE / node(s) had untolerated taint.

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
Observed Symptom: Pod remains in Pending. Events report: 0/N nodes are
available: N node(s) had untolerated taint.

Execute read-only diagnostic commands:
```bash
# 1. Extract pod tolerations from the unschedulable pod
kubectl get pod {pod_name} -n {namespace} -o jsonpath='{.spec.tolerations}'

# 2. Query all taints across all nodes in the cluster
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# 3. Check for GPU/TPU accelerator disruption taints
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.taints[?(@.key=="nvidia.com/gpu")]}{"\n"}{end}'

# 4. Inspect node pool taints via GKE API
gcloud container node-pools list \
  --cluster="{cluster_name}" --region="{location}" --project="{project_id}" \
  --format="table(name,config.taints)"
```

---

### Step 2: Diagnostic Decision Tree
- Dedicated Accelerator Taints: Nodes with GPUs/TPUs carry taints like
  nvidia.com/gpu:NoSchedule. Workload requested GPU resources or runs on GPU
  pool without toleration.
- Tenant Isolation Taints: Node pools dedicated to specific environments carry
  custom taints (e.g., dedicated=batch:NoSchedule). Pod lacks matching
  toleration.
- System Node Taints: Nodes cordoned or tainted with
  node.kubernetes.io/unschedulable:NoSchedule or
  node.cloudprovider.kubernetes.io/uninitialized:NoSchedule.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If workload should run on tainted nodes: Add the corresponding toleration
   under spec.template.spec.tolerations in the Deployment manifest.
2. If node was erroneously tainted: Remove the unwanted taint from the node
   pool using gcloud container node-pools update or kubectl taint nodes.
3. If node is uninitialized: Check cloud provider integration and node
   bootstrap logs.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
