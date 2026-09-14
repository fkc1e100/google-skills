---
name: gcp-compute-quota-troubleshooting
metadata:
  category: Management
description: >-
  Diagnoses Compute Engine and GKE resource quota exhaustion (CPUs, GPUs, In-Use IP Addresses, Local SSDs, Persistent Disks). Use when GKE cluster creation, node pool addition, or autoscaling fails with quota exceeded errors, or when Pantheon reports parseErrorMessageToQuotaError. Don't use for billing cost optimization or budget alert management.
---

# Compute Engine & GKE Resource Quota Troubleshooting Skill

Use this skill to diagnose and resolve resource quota exhaustion events in Google Cloud Platform affecting Google Kubernetes Engine (GKE) clusters and Compute Engine infrastructure. When cluster mutations fail due to insufficient regional quota, this skill calculates the exact shortfall and proposes immediate architectural alternatives alongside formal quota increase requests.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: calculate quota utilization first, then propose gcloud CLI commands or Cloud Console quota increase paths for a human administrator to submit.

## 🔍 Diagnostic Workflow

### Step 0: Context Discovery & Error Extraction

1. **Parameter Discovery**: Extract `project_id`, `region`, `machine_type`, and `node_count` from the error message or prompt.
   - Infer region from cluster location (e.g., zone `asia-southeast1-a` maps to region `asia-southeast1`).
2. **Quota Metric Classification**:
   - Machine vCPU limits → `CPUS`, `PREEMPTIBLE_CPUS`
   - Accelerator limits → `GPUS_ALL_REGIONS`, `NVIDIA_L4_GPUS`, `NVIDIA_A100_GPUS`, `NVIDIA_H100_GPUS`
   - Networking limits → `IN_USE_ADDRESSES`
   - Storage limits → `DISKS_TOTAL_GB`, `LOCAL_SSD_TOTAL_GB`

--------------------------------------------------------------------------------

### Step 1: Query Regional Quota Limits and Current Consumption

Retrieve active limits, current usage, and available headroom:

```bash
# 1. Query all regional quotas for the target region
gcloud compute regions describe "{region}" \
  --project="{project_id}" \
  --format="json(quotas)"

# 2. Specifically filter for the relevant failing quota metric
gcloud compute regions describe "{region}" \
  --project="{project_id}" \
  --format="table(quotas.metric,quotas.usage,quotas.limit)" \
  --flatten="quotas[]" \
  --filter="quotas.metric:('{metric}')"
```

For global accelerator quotas:
```bash
gcloud compute project-info describe \
  --project="{project_id}" \
  --format="table(quotas.metric,quotas.usage,quotas.limit)" \
  --flatten="quotas[]" \
  --filter="quotas.metric:('GPUS_ALL_REGIONS')"
```

--------------------------------------------------------------------------------

### Step 2: Calculate Exact Quota Shortfall

Compute required resource delta:

$$\text{Requested Delta} = \text{Requested Nodes} \times \text{Metric Per Node}$$
$$\text{Shortfall} = (\text{Current Usage} + \text{Requested Delta}) - \text{Quota Limit}$$

#### Example Calculation:
- Target: Add 4 nodes of `g2-standard-8` (1 NVIDIA L4 GPU + 8 vCPUs per node) in `asia-southeast1`.
- Quota `NVIDIA_L4_GPUS`: Limit = 4, Current Usage = 2.
- Requested: 4 GPUs.
- Total required: $2 + 4 = 6$.
- Shortfall: $6 - 4 = 2$ GPUs.

--------------------------------------------------------------------------------

### Step 3: Mitigation & Quota Request Plan

Provide the human operator with two remediation paths:

#### 3a. Immediate Architectural Alternative (No Quota Wait):
1. **Alternative Zone/Region**: Check if adjacent regions possess available quota headroom.
2. **Alternative Machine Family**: If standard vCPU quota is exhausted, evaluate whether Compute Engine E2 or N2D families have independent quota ceilings.

#### 3b. Formal Quota Increase Request Command:
Generate the exact gcloud command and Cloud Console link:

```bash
# Submit automated quota increase request via Service Usage API (if enabled)
gcloud service-usage quotas request-increase \
  --service="compute.googleapis.com" \
  --metric="compute.googleapis.com/{metric}" \
  --unit="1" \
  --value="{new_limit}" \
  --project="{project_id}"
```

**Console Deep Link**:
`https://console.cloud.google.com/iam-admin/quotas?project={project_id}&metric={metric}&region={region}`
