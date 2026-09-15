---
name: gke-control-plane-health
description: >-
  Diagnoses GKE control plane condition degradation, API server request latencies, admission webhook timeouts, and etcd performance issues. Use when the GKE console displays cluster degradation warnings or buildUnknownConditionMessage. Don't use for worker node health (use gke-node-notready-troubleshooting).
---

# GKE Control Plane Health Troubleshooting Skill

## Purpose & Scope
This skill diagnoses cluster lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
buildUnknownConditionMessage / Cluster condition Degraded / API server latency
high.

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
Observed Symptom: GKE console displays cluster degradation warnings. Kubectl
commands experience high latency or timeouts.

Execute read-only diagnostic commands:
```bash
# 1. Inspect cluster conditions from GKE API
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" \
  --format="yaml(conditions,currentMasterVersion)"

# 2. Check validating and mutating admission webhooks in the cluster
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -o wide

# 3. Query Cloud Monitoring for API server request durations
gcloud logging read 'resource.type="k8s_cluster" AND severity>=WARNING' \
  --freshness=1h --limit=20 --project="{project_id}"

# 4. Check for slow or failing admission webhook calls in audit logs
gcloud logging read \
  'protoPayload.serviceName="k8s.io" AND protoPayload.status.code=504' \
  --freshness=1h --limit=10 --project="{project_id}"
```

---

### Step 2: Diagnostic Decision Tree
- Misbehaving Admission Webhook: Third-party webhook (e.g. Istio, Datadog,
  Gatekeeper) timing out, blocking API server request handling.
- Control Plane Scaling / Maintenance: Google Cloud is actively scaling or
  patching control plane VMs.
- High Request Concurrency / Client Throttling: Runaway controller or agent
  spamming API server with list requests without resourceVersion=0.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. Inspect or temporarily disable failing admission webhooks configured with
   failurePolicy: Fail.
2. Optimize client applications spamming API server with unbounded LIST
   calls.
3. If control plane condition persists, contact Google Cloud Support with
   cluster UUID.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
