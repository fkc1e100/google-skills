---
name: gke-pvc-binding-troubleshooting
description: >-
  Diagnoses GKE PersistentVolumeClaims (PVC) stuck in Pending, missing StorageClasses, dynamic volume provisioning failures, or volume mount timeouts. Use when pods cannot start due to unbound PVCs, PVC_NOT_FOUND, or POD_HAS_UNBOUND_IMMEDIATE_PVC. Don't use for compute resource scheduling shortages (use gke-unschedulable-pod-troubleshooting).
---

# GKE PVC Binding Troubleshooting Skill

## Purpose & Scope
This skill diagnoses storage & volumes failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.PVC_NOT_FOUND / IssueCategory.POD_HAS_UNBOUND_IMMEDIATE_PVC.

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
Observed Symptom: Pod remains in Pending or ContainerCreating. PVC status
displays Pending. Events report FailedBinding or FailedMount.

Execute read-only diagnostic commands:
```bash
# 1. Inspect PVC status, requested capacity, and StorageClass
kubectl get pvc {pvc_name} -n {namespace} -o yaml

# 2. Query events related to the PVC
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pvc_name} \
  --sort-by='.metadata.creationTimestamp'

# 3. Inspect cluster StorageClasses and volumeBindingMode
kubectl get storageclass -o custom-columns=\
NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BIND_MODE:.volumeBindingMode

# 4. For volume mount issues: inspect pod volume attachment events
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pod_name} \
  --sort-by='.metadata.creationTimestamp'
```

---

### Step 2: Diagnostic Decision Tree
- StorageClass Not Found: PVC references a storageClassName that does not
  exist in the cluster.
- WaitForFirstConsumer Pending: StorageClass uses volumeBindingMode:
  WaitForFirstConsumer. PVC will remain Pending until a pod consuming it is
  scheduled on a node.
- Zone Mismatch / Multi-Zone Conflict: Persistent disk is located in zone-a,
  but pod was scheduled on a node in zone-b.
- Volume In Use by Another Pod: ReadWriteOnce (RWO) disk attached to an
  existing node and cannot be multi-attached.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If missing StorageClass: Correct storageClassName in the PVC manifest or
   create the required StorageClass.
2. If WaitForFirstConsumer: Inspect why the consuming pod cannot schedule
   (check CPU/memory requests or node selectors).
3. If RWO multi-attach error: Terminate the old pod holding the disk or
   migrate to ReadWriteMany (NFS / Filestore CSI).

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
