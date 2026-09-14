---
name: gke-operation-troubleshooting
metadata:
  category: Containers
description: >-
  Diagnoses asynchronous GKE long-running mutation operation failures (cluster creation, node pool upgrade, resizing, maintenance) and correlates Pantheon MutationErrorBox tracking IDs with backend error logs. Use when GKE asynchronous operations fail, timeout, or return unhandled Coliseum errors. Don't use for in-cluster pod runtime crashes (use gke-workload-troubleshooting).
---

# GKE Mutation Operation Troubleshooting Skill

Use this skill to systematically diagnose why a Google Kubernetes Engine (GKE) asynchronous mutation operation (such as `CREATE_CLUSTER`, `UPDATE_CLUSTER`, `UPGRADE_NODES`, or `DELETE_NODE_POOL`) failed. In the Google Cloud Console, these failures manifest inside the `MutationErrorBox` component accompanied by an error summary and a unique operation tracking number.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: retrieve and decode the operation error status first, then propose a safe corrective action.

## 🔍 Diagnostic Workflow

### Step 0: Context Discovery & Operation Identifier

1. **Parameter Discovery**: Extract `project_id`, `operation_id`, and `location` non-interactively from the user prompt or error banner:
   - Operation ID pattern: `operation-{timestamp}-{hash}` (e.g., `operation-1788740043003-c6053677-3e90-4cd8-8c46-f0fdce32cf6c`).
   - Infer location and project from active `gcloud` settings if omitted.

--------------------------------------------------------------------------------

### Step 1: Inspect Long-Running Operation Record

Query the GKE Operations API to fetch the full status payload and error message:

```bash
# 1. Describe the specific failed operation
gcloud container operations describe "{operation_id}" \
  --location="{location}" \
  --project="{project_id}" \
  --format="yaml"

# 2. List recent failed operations in the project if operation ID is unknown
gcloud container operations list \
  --project="{project_id}" \
  --filter="status:ABORTING OR error.message:*" \
  --limit=5 \
  --format="table(name,type,target,status,statusMessage,startTime)"
```

#### Key Fields to Parse:
- `status`: `DONE` vs `ABORTING` vs `RUNNING`.
- `statusMessage`: The high-level error summary surfaced to the client.
- `error.details`: Structured error proto with Coliseum error codes.

--------------------------------------------------------------------------------

### Step 2: Correlate Tracking ID with Cloud Audit Logs

Search Cloud Logging for the backend RPC transaction corresponding to the operation:

```bash
gcloud logging read '
  resource.type="k8s_cluster"
  log_id("cloudaudit.googleapis.com/activity")
  protoPayload.serviceData.containerEngine.operationName="{operation_id}"
' --project="{project_id}" --format="json"
```

#### Common Failure Classifications:
1. **Compute Engine Admission Rejection**:
   - `ZONE_RESOURCE_POOL_EXHAUSTED`: Stockout of selected machine type in target zone.
   - `QUOTA_EXCEEDED`: Exceeded regional quota during instance creation.
2. **Subnet / CIDR Conflict**:
   - `IP_ADDRESS_RANGE_IS_UNAVAILABLE`: Pod CIDR overlaps with an existing VPC subnet.
3. **IAM Service Account Permission Rejection**:
   - Node service account missing `roles/container.defaultNodeServiceAccount` or `roles/logging.logWriter`.
4. **Mutating Webhook Interception**:
   - Internal webhook rejecting system manifests during cluster initialization.

--------------------------------------------------------------------------------

### Step 3: Synthesis & Remediation Plan

Provide the exact corrective `gcloud` command or manifest revision to unblock the operator:
- **For Stockout (`ZONE_RESOURCE_POOL_EXHAUSTED`)**: Propose creating the node pool in an alternative zone within the region.
- **For CIDR Overlap**: Propose non-overlapping secondary IP ranges for pod and service networks.
- **For Quota Exhaustion**: Direct to the quota increase request workflow via `skills/cloud/gcp-compute-quota-troubleshooting`.
