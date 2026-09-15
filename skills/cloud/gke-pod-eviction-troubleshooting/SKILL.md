---
name: gke-pod-eviction-troubleshooting
description: >-
  Diagnoses GKE pod evictions triggered by node disk pressure, memory pressure, or container ephemeral-storage exhaustion exceeding local limits. Use when pods transition to Failed or Evicted phase with message The node was low on resource. Don't use for normal graceful termination or voluntary pod disruption budgets.
---

# GKE Pod Eviction Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by Pod phase Failed,
Reason Evicted / The node was low on resource.

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
Observed Symptom: Pods terminate with status Reason: Evicted. Node conditions
display DiskPressure or MemoryPressure.

Execute read-only diagnostic commands:
```bash
# 1. Inspect evicted pod details and eviction message
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{.status.phase}{": "}{.status.reason}{" - "}{.status.message}{"\n"}'

# 2. Inspect node conditions for pressure signals on the node where the pod ran
kubectl get nodes {node_name} \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" ("}{.reason}{": "}{.message}{")\n"}{end}'

# 3. Check container ephemeral-storage limits and requests
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .spec.containers[*]}{.name}{" Eph-Storage: "}{.resources.limits.ephemeral-storage}{"\n"}{end}'

# 4. List all evicted pods in the namespace to evaluate fleet blast radius
kubectl get pods -n {namespace} --field-selector status.phase=Failed \
  -o custom-columns=NAME:.metadata.name,REASON:.status.reason,NODE:.spec.nodeName
```

---

### Step 2: Diagnostic Decision Tree
- The node was low on resource: ephemeral-storage: Node root filesystem
  exceeded eviction threshold (default 85% disk usage or < 10% available).
- Container local ephemeral-storage limit exceeded: Container wrote more logs
  or local files than allowed by resources.limits.ephemeral-storage.
- Node MemoryPressure Eviction: Kubelet evicted BestEffort/Burstable pods to
  prevent node kernel panic when memory dropped below memory.available<100Mi.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If container exceeded ephemeral-storage: Specify
   resources.limits.ephemeral-storage and mount a PersistentVolume or EmptyDir
   with size limit.
2. If node disk full: Clean up dangling images, truncate bloated container
   log files, or expand node boot disk size.
3. Remove dead evicted pods using: kubectl delete pods -n {namespace}
   --field-selector status.phase=Failed.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
