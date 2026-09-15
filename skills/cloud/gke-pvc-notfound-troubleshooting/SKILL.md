---
name: gke-pvc-notfound-troubleshooting
description: >-
  Diagnoses GKE workloads blocked by non-existent PersistentVolumeClaims (PVC_NOT_FOUND). Use when pods cannot schedule or start with FailedMount / FailedAttachVolume errors because the referenced PVC does not exist in the target namespace. Don't use for unbound PVCs with missing StorageClasses (use gke-pvc-binding-troubleshooting).
---

# GKE PersistentVolumeClaim Not Found Troubleshooting Skill

## Purpose & Scope
This skill diagnoses storage resolution failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
`IssueCategory.PVC_NOT_FOUND`.

This skill operates non-interactively and enforces a read-only diagnostics
boundary: inspect the workload's volume specifications, verify namespace-scoped
PVC inventory, identify volume claim name typos, and propose declarative
PersistentVolumeClaim manifests for human review before any change reaches
production.

---

## Diagnostic Workflow

### Step 0: Non-Interactive Context Discovery & Time Window
1. Context Extraction: Extract `project_id`, `cluster_name`, `cluster_location`,
   `namespace`, `pod_name`, and referenced volume claim names from the prompt or environment.
2. Time Window: Center a 1-hour query window around the incident
   (`start = issue_time - 30m`, `end = issue_time + 30m`).

---

### Step 1: Physical Symptom & Event Inspection
Observed Symptom: Pod remains in `ContainerCreating` or `Pending`. Events report
`FailedMount` or `FailedAttachVolume` with message `persistentvolumeclaim "<name>" not found`.

Execute read-only diagnostic commands:
```bash
# 1. Inspect pod volume claims and mount specifications
kubectl get pod {pod_name} -n {namespace} -o jsonpath='{range .spec.volumes[*]}{.name}{"\t"}{.persistentVolumeClaim.claimName}{"\n"}{end}'

# 2. Query pod failure events
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pod_name} \
  --sort-by='.metadata.creationTimestamp'

# 3. Check existing PersistentVolumeClaims in the namespace
kubectl get pvc -n {namespace} -o custom-columns=\
NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.volumeName,CAPACITY:.status.capacity.storage,STORAGECLASS:.spec.storageClassName

# 4. Check for similarly named PVCs across adjacent namespaces (detect namespace mismatch)
kubectl get pvc --all-namespaces --field-selector metadata.name={pvc_name}
```

---

### Step 2: Diagnostic Decision Tree
- **PVC Missing in Namespace**: The referenced `claimName` does not exist in the
  workload's namespace. Verify if the PVC was deleted or never applied.
- **Namespace Mismatch**: The PVC exists in another namespace (e.g. `default` or `storage-system`),
  but the workload is deployed in `{namespace}`. PVCs are strictly namespace-scoped.
- **Typo in Volume Claim Name**: Workload specification contains an invalid or
  misspelled `claimName` (e.g. `data-vol-pvc` vs `datavol-pvc`).
- **StatefulSet VolumeClaimTemplate Drift**: Pod is part of a StatefulSet whose
  volumeClaimTemplate generated a predictable PVC name that was manually deleted.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. **Missing PVC**: Generate a standard `PersistentVolumeClaim` manifest with the
   cluster's default StorageClass (`standard-rwo`) and requested storage capacity.
2. **Namespace Mismatch**: Propose either redeploying the workload to the PVC's
   namespace or creating a corresponding PVC in the workload's namespace.
3. **Typo in Workload Spec**: Provide a GitOps patch correcting `spec.volumes[].persistentVolumeClaim.claimName`
   in the Deployment or StatefulSet manifest.

### Operational Guarantees
- **Read-Only Verification**: All diagnostic queries use read-only inspection commands (`kubectl get`).
- **No Autonomous Deletion**: Never delete or unbind persistent storage volumes autonomously.
