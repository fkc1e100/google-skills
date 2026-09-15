---
name: gke-cluster-provisioning-troubleshooting
description: >-
  Diagnoses GKE cluster creation and provisioning timeouts, invalid network configurations, private cluster control plane peering issues, and service account IAM permission gaps. Use when a new GKE cluster fails to provision or transitions to ERROR state during creation. Don't use for existing cluster upgrades (use gke-cluster-upgrade-troubleshooting).
---

# GKE Cluster Provisioning Troubleshooting Skill

## Purpose & Scope
This skill diagnoses cluster lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by Cluster creation
failure / Status ERROR / Provisioning timeout.

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
Observed Symptom: Cluster creation operation terminates with code ERROR. Cluster
status shows ERROR or DEGRADED.

Execute read-only diagnostic commands:
```bash
# 1. Describe cluster status and conditions
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" \
  --format="yaml(status,statusMessage,conditions)"

# 2. Check cluster creation long-running operation details
gcloud container operations list --region="{location}" \
  --filter="operationType=CREATE_CLUSTER" --limit=5 --project="{project_id}"

# 3. Check Google Kubernetes Engine service agent IAM permissions
gcloud projects get-iam-policy "{project_id}" \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/container.serviceAgent" \
  --format="table(bindings.role,bindings.members)"

# 4. Check VPC subnet and firewall rules for private cluster control plane peering
gcloud compute networks subnets describe "{subnet_name}" \
  --region="{region}" --project="{project_id}" \
  --format="yaml(privateIpGoogleAccess)"
```

---

### Step 2: Diagnostic Decision Tree
- Service Agent Permission Missing: service-{project_number}@container-engine-
  robot.iam.gserviceaccount.com lacks roles/container.serviceAgent.
- Private Cluster Control Plane CIDR Conflict: Control plane IPv4 CIDR block
  (/28) overlaps with existing VPC subnets or on-prem routes.
- Subnet Private Google Access Disabled: Private cluster nodes cannot reach
  Google APIs because Private Google Access is disabled on subnet.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Grant roles/container.serviceAgent to the GKE service robot account.
2. Ensure control plane CIDR range does not overlap with any VPC subnet
   range.
3. Enable Private Google Access on the target subnet: gcloud compute networks
   subnets update {subnet} --enable-private-ip-google-access.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
