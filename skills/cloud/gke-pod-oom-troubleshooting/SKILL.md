---
name: gke-pod-oom-troubleshooting
description: >-
  Diagnoses GKE pod terminations caused by memory exhaustion, container cgroup limit violations (exit code 137, OOMKilled), and node kernel out-of-memory killing. Use when container statuses display OOMKilled or termination reason OOMKilled. Don't use for CPU throttling or scheduling resource shortages (use gke-unschedulable-pod-troubleshooting).
---

# GKE Pod OOMKilled Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
message.startsWith('OOMKilled') / exitCode: 137.

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
Observed Symptom: Container terminates abruptly with exit code 137 and reason
OOMKilled, or node logs report kernel invocation of oom-killer.

Execute read-only diagnostic commands:
```bash
# 1. Verify OOMKilled status and memory limit on the affected pod
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .status.containerStatuses[*]}{"Container: "}{.name}{"\nTerminated: "}{.lastState.terminated.reason}{"\nExitCode: "}{.lastState.terminated.exitCode}{"\n"}{end}'

# 2. Inspect configured memory requests and limits
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .spec.containers[*]}{.name}{" Requests: "}{.resources.requests.memory}{" Limits: "}{.resources.limits.memory}{"\n"}{end}'

# 3. Check node kernel dmesg / Cloud Logging for kernel-level OOM kills
gcloud logging read 'resource.type="k8s_node" AND textPayload:"Out of memory"' \
  --freshness=1h --limit=10 --project="{project_id}"

# 4. Check historical container memory consumption metrics via Cloud Monitoring
gcloud logging read \
  'resource.type="k8s_container" AND resource.labels.pod_name="{pod_name}" AND severity>=WARNING' \
  --freshness=1h --limit=20 --project="{project_id}"
```

---

### Step 2: Diagnostic Decision Tree
- Container Cgroup OOM: lastState.terminated.reason == 'OOMKilled' while node
  remains healthy. Container resident memory exceeded resources.limits.memory.
- Node-Level Kernel OOM: The node ran out of memory, triggering the Linux
  kernel OOM killer which selected pods according to their oom_score_adj
  (BestEffort before Burstable before Guaranteed).
- JVM / Runtime Heap Misconfiguration: Max heap (-Xmx) set higher than or
  equal to container memory limit, leaving insufficient headroom for JVM off-
  heap / native memory.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If container cgroup limit reached: Increase resources.limits.memory and
   resources.requests.memory in the Deployment manifest.
2. If memory leak detected: Profile application memory allocations and fix
   unbounded caching or memory retention.
3. For Java workloads: Ensure -XX:MaxRAMPercentage=75.0 is configured so JVM
   respects container memory boundaries.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
