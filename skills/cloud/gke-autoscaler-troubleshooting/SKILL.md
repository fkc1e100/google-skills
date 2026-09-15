---
name: gke-autoscaler-troubleshooting
description: >-
  Diagnoses GKE Cluster Autoscaler and Node Auto-Provisioning (NAP) failures to scale up or down, including noDecisionStatus.noScaleUp, max node limits, and unevictable pods blocking scale down. Use when nodes do not scale up to accommodate pending pods or do not scale down idle nodes. Don't use for Horizontal Pod Autoscaler (HPA) workload scaling.
---

# GKE Autoscaler Troubleshooting Skill

## Purpose & Scope
This skill diagnoses scaling & capacity failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
noDecisionStatus.noScaleUp / Cluster autoscaler cannot scale up node pool.

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
Observed Symptom: Pending pods remain unscheduled despite autoscaler enabled.
Node pool does not expand.

Execute read-only diagnostic commands:
```bash
# 1. Inspect cluster autoscaler status ConfigMap in kube-system
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# 2. Check cluster autoscaler logs in Cloud Logging for scale-up rejection reasons
gcloud logging read 'resource.type="k8s_cluster" AND logName:"cluster-autoscaler"' \
  --freshness=1h --limit=20 --project="{project_id}"

# 3. Inspect node pool autoscaling limits
gcloud container node-pools describe "{nodepool_name}" \
  --cluster="{cluster_name}" --region="{location}" --project="{project_id}" \
  --format="yaml(autoscaling)"

# 4. Check pods preventing node scale down
kubectl get pods -A -o wide | grep -i "kube-system"
```

---

### Step 2: Diagnostic Decision Tree
- Max Nodes Limit Reached: Node pool has reached maxNodeCount. Autoscaler
  cannot add more nodes.
- Compute Quota Exceeded: Autoscaler attempted to create instances, but
  Compute Engine rejected creation due to quota.
- Unevictable Pods Blocking Scale Down: Pods without controller (bare pods),
  pods with local storage, or strict PDBs prevent node scale-down.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Increase maxNodeCount on the node pool via gcloud container node-pools
   update --max-nodes.
2. Check and increase Compute Engine regional quotas for CPUs and in-use IP
   addresses.
3. Annotate unevictable pods with 'cluster-autoscaler.kubernetes.io/safe-to-
   evict': 'true' if safe to terminate.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
