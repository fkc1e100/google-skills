---
name: gke-fleet-connect-troubleshooting
metadata:
  category: Containers
description: >-
  Diagnoses Google Kubernetes Engine (GKE) Fleet and Connect registration failures, Connect Agent pod crashes in namespace gke-connect, and Workload Identity federation bindings. Use when cluster registration fails, membership status shows disconnected, or Pantheon reports CLUSTER_DETAIL_REGISTRATION_ERROR. Don't use for general application workload crashes (use gke-workload-troubleshooting).
---

# GKE Fleet & Connect Registration Troubleshooting Skill

Use this skill to systematically diagnose why a Google Kubernetes Engine (GKE) cluster fails to register with Google Cloud Fleet (GKE Hub) or why the Connect Agent cannot establish a secure connection back to Google Cloud.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: gather evidence first, then propose remediations (such as corrected IAM bindings, Workload Identity annotations, or fleet registration commands) for a human operator to apply. **Never** mutate cluster resources or modify IAM policies automatically.

## 🔍 Diagnostic Workflow

### Step 0: Non-Interactive Context Discovery & Time Window

1. **Parameter Discovery**: Extract `project_id`, `cluster_name`, `cluster_location`, and `membership_name` non-interactively from the user prompt or active environment:
   - Default `membership_name` to `cluster_name` if omitted.
   - Infer missing cluster parameters from active environment (`kubectl config current-context` or `gcloud config get-value project`).
2. **Time Window**: Center a 1-hour query window around `{issue_time}` (`start = issue_time - 30m`, `end = issue_time + 30m`).

--------------------------------------------------------------------------------

### Step 1: Inspect Fleet Membership and Registration State

Query Google Cloud Fleet API to evaluate registration status and synchronization health:

```bash
# 1. Describe Fleet Membership in GKE Hub
gcloud container fleet memberships describe "{membership_name}" \
  --project="{project_id}" \
  --format="yaml(name,endpoint,state)"

# 2. Check if Fleet features (e.g., Workload Identity, Config Sync) are enabled
gcloud container fleet features list --project="{project_id}"
```

#### Diagnostic Decision Tree:
- **Membership Not Found (`NOT_FOUND`)**:
  - The cluster registration was never initialized or was deleted. Proceed to **Step 4a (Cluster Registration Remediation)**.
- **State: `DISCONNECTED` or `CONNECT_ERROR`**:
  - The in-cluster Connect Agent cannot establish a bi-directional tunnel to `gkeconnect.googleapis.com`. Proceed to **Step 2 (Connect Agent Pod Health)**.
- **State: `READY` but Console reports error**:
  - Check IAM permissions and console user viewing permissions: `roles/gkehub.viewer`.

--------------------------------------------------------------------------------

### Step 2: Audit In-Cluster Connect Agent Pod Health

Inspect the Connect Agent deployed in namespace `gke-connect`:

```bash
# 1. Check Connect Agent Pod and Deployment status
kubectl get pods,deployment -n gke-connect -o wide

# 2. Check Connect Agent container termination and logs
kubectl logs -n gke-connect -l app=gke-connect-agent --tail=100
```

#### Common Failure Patterns:
- **`ImagePullBackOff` / `ErrImagePull`**:
  - Connect agent image tag is invalid or egress to `gcr.io/gkeconnect` is blocked by VPC firewall.
- **`CrashLoopBackOff` with `permission denied` or `token exchange failed`**:
  - Workload Identity Federation failure between Kubernetes ServiceAccount (`gke-connect-agent`) and Google ServiceAccount. Proceed to **Step 3 (Workload Identity & IAM Audit)**.
- **Dial error to `gkeconnect.googleapis.com:443`**:
  - Firewall egress rule blocking TCP port 443 from cluster nodes to Google APIs.

--------------------------------------------------------------------------------

### Step 3: Validate Workload Identity & IAM Bindings

Inspect the identity federation parameters enabling the Connect Agent to authenticate:

```bash
# 1. Inspect KSA annotations in namespace gke-connect
kubectl get serviceaccount gke-connect-agent -n gke-connect -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'

# 2. Check Google Service Account IAM Policy Bindings
gcloud iam service-accounts get-iam-policy "{gsa_email}" --project="{project_id}" --format="json"

# 3. Verify Project-Level Roles on the Connect GSA
gcloud projects get-iam-policy "{project_id}" \
  --flatten="bindings[].members" \
  --filter="bindings.members:{gsa_email}" \
  --format="table(bindings.role)"
```

#### Verification Criteria:
1. **Workload Identity User**: Ensure `roles/iam.workloadIdentityUser` is bound to:
   `serviceAccount:{project_id}.svc.id.goog[gke-connect/gke-connect-agent]`
2. **Fleet Connect Role**: Ensure the GSA holds:
   `roles/gkehub.connect`
   `roles/gkehub.viewer`

--------------------------------------------------------------------------------

### Step 4: Remediation Plan

Provide the exact corrective actions for the human operator:

#### 4a. Cluster Re-Registration Command:
```bash
gcloud container fleet memberships register "{membership_name}" \
  --gke-cluster="{cluster_location}/{cluster_name}" \
  --enable-workload-identity \
  --project="{project_id}"
```

#### 4b. Fix Workload Identity IAM Binding:
```bash
gcloud iam service-accounts add-iam-policy-binding "{gsa_email}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:{project_id}.svc.id.goog[gke-connect/gke-connect-agent]" \
  --project="{project_id}"
```
