---
name: gke-control-plane-health
metadata:
  category: Containers
description: >-
  Diagnoses GKE control plane health, API server latency, etcd responsiveness, master maintenance status, and unhandled cluster conditions. Use when GKE clusters display degraded health banners, API server connection timeouts, or when Pantheon reports buildUnknownConditionMessage. Don't use for pod-level application errors (use gke-workload-troubleshooting).
---

# GKE Control Plane & Cluster Health Troubleshooting Skill

Use this skill to diagnose and resolve GKE control plane degradations, master component unresponsiveness, API server request latencies, and cluster condition alerts. The control plane is fully managed by Google in GKE, but master health can be impaired by heavy mutating admission webhooks, saturated etcd keyspaces, maintenance operations, or network firewall blocks.

This skill operates **non-interactively** and enforces a **read-only diagnostics boundary**: audit control plane telemetry first, then propose remediations for an administrator to apply.

## 🔍 Diagnostic Workflow

### Step 0: Context Discovery & Time Window

1. **Parameter Discovery**: Extract `project_id`, `cluster_name`, and `cluster_location` non-interactively from prompt or active environment (`kubectl config current-context`, `gcloud config get-value project`).
2. **Time Window**: Center a 1-hour query window around `{issue_time}` (`start = issue_time - 30m`, `end = issue_time + 30m`).

--------------------------------------------------------------------------------

### Step 1: Inspect Cluster Conditions & Master Status

Query the GKE API to inspect active cluster conditions and control plane versions:

```bash
# 1. Describe cluster status and active conditions
gcloud container clusters describe "{cluster_name}" \
  --location="{cluster_location}" \
  --project="{project_id}" \
  --format="yaml(status,statusMessage,conditions,currentMasterVersion,currentNodeVersion)"

# 2. Check if cluster is in RECONCILING, DEGRADED, or ERROR state
```

#### Common Condition Codes:
- `ClusterConditionCode: CA_NO_DECISION`: Autoscaler has encountered an internal decision error.
- `ClusterConditionCode: SATURATED`: Control plane experiencing severe load shedding.
- `ClusterConditionCode: MASTER_UPGRADING`: Expected control plane version bump in progress.

--------------------------------------------------------------------------------

### Step 2: Audit Cloud Audit Logs for Master Operations

Check whether Google or a cluster administrator recently initiated a control plane operation:

```bash
gcloud logging read '
  resource.type="k8s_cluster"
  resource.labels.cluster_name="{cluster_name}"
  resource.labels.location="{cluster_location}"
  protoPayload.serviceName="container.googleapis.com"
' --project="{project_id}" --limit=20 --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail)"
```

#### Diagnostic Decision:
- If a `SetMasterAuth`, `UpdateCluster`, or `UpgradeMaster` operation is active, the control plane is undergoing planned maintenance. Advise the user on expected completion windows rather than treating as an outage.

--------------------------------------------------------------------------------

### Step 3: Inspect Kubernetes Webhook & API Latency Sinks

Misconfigured mutating or validating admission webhooks are the leading cause of control plane timeouts:

```bash
# 1. List all validating and mutating webhooks
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations

# 2. Check webhooks with failurePolicy=Fail and non-responsive endpoints
kubectl get mutatingwebhookconfigurations -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.webhooks[*].failurePolicy}{"\n"}{end}'
```

#### Rule:
A webhook configured with `failurePolicy: Fail` whose serving pod is down will reject all API server mutations cluster-wide.

--------------------------------------------------------------------------------

### Step 4: Remediation Plan

Generate actionable mitigations:
1. **Webhook Bypass / Temporary Permissive Mode**:
   If a broken admission webhook is blocking API server operations, propose switching `failurePolicy` from `Fail` to `Ignore`:
   ```bash
   kubectl patch mutatingwebhookconfiguration "{webhook_name}" --type='json' -p='[{"op": "replace", "path": "/webhooks/0/failurePolicy", "value": "Ignore"}]'
   ```
2. **Master Authorized Networks**:
   Verify that the administrator's client IP or Cloud Shell range is authorized in `masterAuthorizedNetworksConfig`.
