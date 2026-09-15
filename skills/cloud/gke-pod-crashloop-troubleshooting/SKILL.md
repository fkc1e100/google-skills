---
name: gke-pod-crashloop-troubleshooting
description: >-
  Diagnoses GKE container CrashLoopBackOff states, application process crashes, and probe failure loops. Use when a pod container restarts repeatedly, terminates with non-zero exit codes, or reports CrashLoopBackOff in the GKE console. Don't use for image pull errors (use gke-image-pull-troubleshooting) or memory limit terminations (use gke-pod-oom-troubleshooting).
---

# GKE Pod CrashLoopBackOff Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.CONTAINER_KEEPS_CRASHING / IssueDetail.Type.CRASH_LOOP_BACKOFF.

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
Observed Symptom: Container in pod restarts repeatedly, entering
CrashLoopBackOff with increasing back-off delays (10s, 20s, 40s up to 5m).

Execute read-only diagnostic commands:
```bash
# 1. Inspect pod container statuses and termination reason
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{": "}{.state}{" lastState: "}{.lastState}{" restartCount: "}{.restartCount}{"\n"}{end}'

# 2. Inspect previous terminated container exit code and termination message
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{.status.containerStatuses[?(@.name=="{container_name}")].lastState.terminated}'

# 3. Retrieve standard output and error logs of the previous terminated container
kubectl logs {pod_name} -n {namespace} -c {container_name} --previous --tail=100

# 4. Check pod events for liveness/readiness probe failures or timeouts
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pod_name} \
  --sort-by='.metadata.creationTimestamp'
```

---

### Step 2: Diagnostic Decision Tree
- ExitCode 1 or General Error: Application code threw an unhandled exception,
  syntax error, or missing configuration. Check previous logs for stack traces.
- ExitCode 137 (OOMKilled): Terminated by SIGKILL due to exceeding container
  memory limit. Note: Route to gke-pod-oom-troubleshooting if memory limit is
  root cause.
- ExitCode 139 (Segmentation Fault) or 143 (SIGTERM): Process crashed on
  memory access violation or was terminated by orchestration signal.
- Probe Failure Loop (Liveness probe failed): Container starts, but liveness
  probe fails consecutively, causing kubelet to kill and restart the container.
  Check probe initialDelaySeconds, periodSeconds, and endpoint HTTP response
  status.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If missing environment variable or configuration: Update the ConfigMap or
   Secret referenced by env / envFrom in the Deployment manifest.
2. If liveness probe timing issue: Increase initialDelaySeconds or
   failureThreshold in the pod spec to accommodate application startup time.
3. If entrypoint command failure: Verify command and args in the container
   spec against the container image entrypoint.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
