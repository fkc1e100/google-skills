---
name: gcp-compute-quota-troubleshooting
description: >-
  Diagnoses Compute Engine resource quota exhaustion blocking GKE cluster provisioning, node pool additions, or horizontal autoscaling. Use when cluster operations, node pool creation, or node expansions fail with quota exceeded errors (CPUS, GPUS_ALL_REGIONS, DISKS_TOTAL_GB). Don't use for application memory limits (use gke-pod-oom-troubleshooting).
---

# GCP Compute Engine Quota Troubleshooting Skill

## Purpose & Scope
This skill diagnoses platform & quota failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by
parseErrorMessageToQuotaError / Quota exceeded for resource.

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
Observed Symptom: Node pool creation fails, node pool cannot scale up, or GKE
cluster creation fails with Quota exceeded for metric.

Execute read-only diagnostic commands:
```bash
# 1. Describe regional Compute Engine quotas in target region
gcloud compute regions describe "{region}" \
  --project="{project_id}" --format="yaml(quotas)"

# 2. Query project-level global Compute Engine quotas
gcloud compute project-info describe \
  --project="{project_id}" --format="yaml(quotas)"

# 3. Inspect recent failed operations in Compute Engine for quota details
gcloud compute operations list --regions="{region}" \
  --filter="status=DONE AND error:*" --limit=10 --project="{project_id}"

# 4. Check GKE cluster operation error messages for quota limit details
gcloud container operations list --region="{region}" \
  --filter="status=DONE AND error:*" --limit=5 --project="{project_id}"
```

---

### Step 2: Diagnostic Decision Tree
- CPUS or CPUS_ALL_REGIONS Exceeded: Total requested vCPUs for the node pool
  exceeds regional or global quota limit.
- NVIDIA_A100_GPUS / GPUS_ALL_REGIONS Exceeded: Workload requires GPU
  accelerators that exceed allocated project quota.
- IN_USE_ADDRESSES Exceeded: Cluster or node pool requires more
  external/internal IP addresses than permitted by project network quota.
- SSD_TOTAL_GB / DISKS_TOTAL_GB Exceeded: Boot disks or attached
  PersistentVolumes exceed disk storage quota in the region.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Request quota increase via Google Cloud Console (IAM & Admin -> Quotas) or
   gcloud services quota request.
2. Switch node pool machine type to an alternate family with available quota
   (e.g. from n2-standard to c3-standard).
3. Delete unused disks or scale down non-production node pools to reclaim
   committed quota.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
