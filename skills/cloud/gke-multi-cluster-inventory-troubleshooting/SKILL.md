---
name: gke-multi-cluster-inventory-troubleshooting
metadata:
  category: Containers
description: >-
  Diagnoses partial inventory fetch failures and RPC timeouts across multi-cluster fleet consoles in Google Cloud. Use when cluster, workload, or node inventory list views report partial load errors, ErrorType.CLUSTER_WIDE, or ErrorType.NAMESPACE_LEVEL. Don't use for single-cluster application pod crashes (use gke-workload-troubleshooting).
---

# GKE Multi-Cluster Inventory Troubleshooting Skill

Use this skill to diagnose and resolve partial inventory aggregation failures across multi-cluster fleet consoles in Google Cloud. When an operator views cluster or workload list pages across multiple regions and clusters, Pantheon's `PartialErrorComponent` surfaces alerts when one or more clusters fail to return their resource inventory within the RPC deadline.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: inspect connectivity, credentials, and control plane reachability without mutating live cluster resources.

## 🔍 Diagnostic Workflow

### Step 0: Context Discovery & Project Inventory

1. **Parameter Discovery**: Extract `project_id` and list of unreachable clusters from the error banner or prompt.
2. **List All Registered Clusters in Fleet / Project**:
   ```bash
   gcloud container clusters list --project="{project_id}" --format="table(name,location,status,endpoint)"
   ```

--------------------------------------------------------------------------------

### Step 1: Classify Partial Failure Mode

Inspect the specific failure type reported:
- **`CLUSTER_WIDE`**: The entire cluster endpoint is unreachable; no namespace, node, or workload data could be retrieved.
- **`NAMESPACE_LEVEL`**: The cluster responded, but specific namespaces timed out or were rejected by RBAC.
- **`AUTOPILOT_CLUSTER_WITH_ZERO_NODES`**: A newly provisioned Autopilot cluster has not yet scheduled any workloads, so zero compute nodes exist.

--------------------------------------------------------------------------------

### Step 2: Diagnose Network Reachability to Cluster Control Plane

For `CLUSTER_WIDE` failures, verify whether the management plane can reach the Kubernetes API endpoint:

```bash
# 1. Check if the cluster is a Private Cluster
gcloud container clusters describe "{cluster_name}" \
  --location="{cluster_location}" \
  --project="{project_id}" \
  --format="yaml(privateClusterConfig)"

# 2. Check Master Authorized Networks
gcloud container clusters describe "{cluster_name}" \
  --location="{cluster_location}" \
  --project="{project_id}" \
  --format="yaml(masterAuthorizedNetworksConfig)"
```

#### Diagnostic Decision:
- If `enablePrivateEndpoint: true` and Google Cloud Console access is not federated through Connect Agent, browser requests to the cluster IP will fail because the endpoint is private to the VPC.
- If `masterAuthorizedNetworksConfig.enabled: true` and public access is disabled, ensure the management CIDRs are allowlisted.

--------------------------------------------------------------------------------

### Step 3: Audit Fleet Connect Status for Remote Clusters

If the cluster is registered to Google Cloud Fleet, query the Connect gateway health:

```bash
gcloud container fleet memberships describe "{cluster_name}" \
  --project="{project_id}" \
  --format="yaml(state.code,state.description)"
```

--------------------------------------------------------------------------------

### Step 4: Remediation Plan

Provide clear architectural guidance:
1. **Private Cluster Console Access**: Propose enabling the Connect Gateway (`gcloud container fleet memberships register ... --enable-workload-identity`) so the Google Cloud Console can securely tunnel requests through Google's backbone without exposing public endpoints.
2. **Master Authorized Networks**: Propose adding Cloud Shell or authorized administrator IP blocks to `masterAuthorizedNetworksConfig`.
