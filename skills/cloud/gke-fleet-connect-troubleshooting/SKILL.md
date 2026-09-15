---
name: gke-fleet-connect-troubleshooting
description: >-
  Diagnoses GKE Fleet (GKE Hub) cluster registration errors, Connect Agent pod crashes in namespace gke-connect, and Workload Identity federation bindings. Use when cluster registration fails, membership status is disconnected, or the console reports CLUSTER_DETAIL_REGISTRATION_ERROR. Don't use for application workload failures (use gke-pod-crashloop-troubleshooting).
---

# GKE Fleet & Connect Registration Troubleshooting Skill

## Purpose & Scope
This skill diagnoses fleet & hub failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by
CLUSTER_DETAIL_REGISTRATION_ERROR / Fleet membership disconnected.

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
Observed Symptom: Cluster details banner displays registration error. Fleet
membership status is DISCONNECTED or CONNECT_ERROR.

Execute read-only diagnostic commands:
```bash
# 1. Describe Fleet Membership in GKE Hub
gcloud container fleet memberships describe "{membership_name}" \
  --project="{project_id}" --format="yaml(name,endpoint,state)"

# 2. Inspect Connect Agent pods in namespace gke-connect
kubectl get pods -n gke-connect -o wide

# 3. Check Connect Agent logs for authentication or dial errors
kubectl logs -n gke-connect -l app=gke-connect-agent --tail=100

# 4. Validate Workload Identity binding on Connect Agent service account
kubectl get serviceaccount gke-connect-agent -n gke-connect \
  -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'
```

---

### Step 2: Diagnostic Decision Tree
- Membership Not Registered: Cluster was never registered or registration was
  removed.
- Connect Agent Pod Crashing: Connect agent image pull error or token exchange
  failure.
- Workload Identity Binding Missing: roles/iam.workloadIdentityUser is not
  bound to gke-connect-agent KSA.
- Egress Port 443 Blocked: Cluster nodes cannot dial
  gkeconnect.googleapis.com:443 through VPC firewall.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Re-register cluster using gcloud container fleet memberships register
   {cluster_name} --gke-cluster={location}/{cluster_name} --enable-workload-
   identity.
2. Ensure Google Service Account holds roles/gkehub.connect and
   roles/gkehub.viewer.
3. Allow outbound HTTPS (TCP port 443) to Google APIs in VPC firewall rules.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
