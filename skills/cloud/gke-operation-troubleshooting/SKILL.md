---
name: gke-operation-troubleshooting
description: >-
  Diagnoses asynchronous GKE mutation operation failures across cluster updates, node pool resizes, and configuration mutations. Use when operations display status DONE with error details or the GKE console displays MutationErrorBox. Don't use for synchronous Kubernetes API errors.
---

# GKE Mutation Operation Troubleshooting Skill

## Purpose & Scope
This skill diagnoses platform & quota failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by MutationErrorBox /
hasOtherError / Operation status DONE with error.

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
Observed Symptom: An asynchronous GKE operation (cluster update, node pool
resize, feature enablement) fails with error message.

Execute read-only diagnostic commands:
```bash
# 1. Describe the specific failed GKE operation
gcloud container operations describe "{operation_id}" \
  --location="{location}" --project="{project_id}" --format="yaml"

# 2. List recent failed GKE operations in the project
gcloud container operations list --filter="status=DONE AND error:*" \
  --limit=10 --project="{project_id}"

# 3. Query Cloud Audit Logs for initiating API call and error envelope
gcloud logging read \
  'resource.type="gke_cluster" AND protoPayload.serviceData.containerEngine:*' \
  --freshness=2h --limit=10 --project="{project_id}"
```

---

### Step 2: Diagnostic Decision Tree
- Resource Dependency Conflict: Operation failed because dependent resource
  (network, subnet, or route) was in use or deleted.
- Invalid Feature Flag Combination: Configuration request attempted mutually
  exclusive features (e.g. legacy networking with Dataplane V2).
- Backend Service Agent Timeout: Underlying Compute Engine instance group
  manager failed to converge within operation timeout.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Review the full detail field in the operation response for actionable
   error codes.
2. Re-run failed configuration update after correcting mutually exclusive
   parameters.
3. Clean up dangling instance group managers if node pool operation failed.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
