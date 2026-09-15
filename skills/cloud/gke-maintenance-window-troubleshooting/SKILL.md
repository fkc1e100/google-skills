---
name: gke-maintenance-window-troubleshooting
description: >-
  Diagnoses GKE cluster maintenance window and exclusion policy misconfigurations, blocked security patches, and unexpected maintenance operations. Use when maintenance policies prevent required cluster updates or auto-upgrades trigger unexpectedly. Don't use for general cluster upgrade errors (use gke-cluster-upgrade-troubleshooting).
---

# GKE Maintenance Window Troubleshooting Skill

## Purpose & Scope
This skill diagnoses cluster lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by Maintenance
exclusion conflict / Maintenance policy preventing upgrade.

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
Observed Symptom: Urgent security patches cannot be applied, or cluster upgrades
occur at unexpected hours.

Execute read-only diagnostic commands:
```bash
# 1. Describe cluster maintenance policy and exclusion windows
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" \
  --format="yaml(maintenancePolicy)"

# 2. Inspect cluster notification settings for upgrade events
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" \
  --format="yaml(notificationConfig)"

# 3. Check recent maintenance and auto-upgrade audit logs
gcloud logging read \
  'protoPayload.methodName:"google.container.v1.ClusterManager.UpdateCluster"' \
  --freshness=7d --limit=10 --project="{project_id}"
```

---

### Step 2: Diagnostic Decision Tree
- Continuous Maintenance Exclusion: Exclusion window configured for > 30 days
  or scope set to NO_UPGRADES blocking mandatory security rollouts.
- Timezone Mismatch in Window: Maintenance window start time configured in UTC
  while team expected local timezone.
- End of Life (EOL) Forced Upgrade: Kubernetes version reached Google Cloud
  end-of-support; GKE overrides exclusions to maintain cluster security.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Adjust maintenance window start time and duration (minimum 4 hours) in
   UTC.
2. Remove or narrow maintenance exclusion scopes before versions reach end-
   of-support.
3. Enable Cloud Pub/Sub upgrade notifications to receive advance warning of
   scheduled upgrades.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
