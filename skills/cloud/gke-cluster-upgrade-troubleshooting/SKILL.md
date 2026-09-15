---
name: gke-cluster-upgrade-troubleshooting
description: >-
  Diagnoses GKE control plane and node pool upgrade failures, maintenance exclusion window conflicts, surge upgrade node quota exhaustion, and deprecated Kubernetes API usage blocks. Use when cluster or node pool upgrades fail, stall, or roll back. Don't use for initial cluster creation (use gke-cluster-provisioning-troubleshooting).
---

# GKE Cluster Upgrade Troubleshooting Skill

## Purpose & Scope
This skill diagnoses cluster lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.UPGRADE_FAILURE / Node pool upgrade stalled / Upgrade rollback.

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
Observed Symptom: Node pool upgrade fails or gets stuck. Nodes fail to drain or
surge instances cannot be provisioned.

Execute read-only diagnostic commands:
```bash
# 1. Check GKE upgrade operations history and error details
gcloud container operations list --region="{location}" \
  --filter="operationType=(UPGRADE_MASTER OR UPGRADE_NODES)" \
  --limit=5 --project="{project_id}"

# 2. Inspect node pool upgrade settings (surge vs blue-green)
gcloud container node-pools describe "{nodepool_name}" \
  --cluster="{cluster_name}" --region="{location}" --project="{project_id}" \
  --format="yaml(upgradeSettings)"

# 3. Check for blocking PodDisruptionBudgets (PDB) preventing node drain
kubectl get pdb -A

# 4. Inspect deprecated Kubernetes API usage blockers in cluster
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" --format="yaml(conditions)"
```

---

### Step 2: Diagnostic Decision Tree
- Surge Node Quota Exhaustion: Surge upgrade requires extra compute quota
  (e.g. maxSurge=1), but regional quota is exhausted.
- Strict PDB Blocking Node Drain: PodDisruptionBudget has minAvailable equal
  to total replicas, preventing kubelet eviction during node drain.
- Deprecated API Blocker: GKE safety checks block automatic minor version
  upgrade due to active usage of deprecated APIs.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Temporarily relax restrictive PDBs during node pool maintenance windows.
2. Configure blue-green upgrade strategy with custom drain timeouts.
3. Migrate deprecated API versions in application manifests before initiating
   control plane upgrade.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
