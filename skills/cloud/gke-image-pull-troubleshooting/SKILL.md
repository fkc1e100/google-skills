---
name: gke-image-pull-troubleshooting
description: >-
  Diagnoses GKE image pull failures including ImagePullBackOff, ErrImagePull, and image inspection errors. Use when pods fail to pull container images from Artifact Registry, Google Container Registry, or external registries due to authentication, missing tags, or network policy restrictions. Don't use for application runtime crashes (use gke-pod-crashloop-troubleshooting).
---

# GKE Image Pull Failure Troubleshooting Skill

## Purpose & Scope
This skill diagnoses workload lifecycle failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by
IssueCategory.CANNOT_PULL_IMAGE / IssueDetail.Type.IMAGE_PULL_BACKOFF.

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
Observed Symptom: Pod remains in Waiting / ImagePullBackOff status, unable to
download container image from registry.

Execute read-only diagnostic commands:
```bash
# 1. Inspect image pull failure events and exact container image URI
kubectl get pod {pod_name} -n {namespace} \
  -o jsonpath='{range .status.containerStatuses[*]}{.name}{": "}{.image}{" -> "}{.state.waiting.reason}{": "}{.state.waiting.message}{"\n"}{end}'

# 2. Inspect events on the affected pod for image pull error strings
kubectl get events -n {namespace} \
  --field-selector involvedObject.name={pod_name} \
  --sort-by='.metadata.creationTimestamp'

# 3. For Artifact Registry images: verify image exists in Artifact Registry
gcloud artifacts docker images list \
  {registry_location}-docker.pkg.dev/{project_id}/{repository}/{image_name} \
  --include-tags --project="{project_id}"

# 4. Check node service account IAM permissions for Artifact Registry reader
gcloud projects get-iam-policy "{project_id}" \
  --flatten="bindings[].members" \
  --filter="bindings.members:{node_service_account}" \
  --format="table(bindings.role)"
```

---

### Step 2: Diagnostic Decision Tree
- HTTP 404 / manifest unknown: The image tag or repository path does not exist
  in Artifact Registry. Verify image name spelling and tag existence.
- HTTP 403 Forbidden / Access Denied: The node service account lacks
  roles/artifactregistry.reader on the target repository or project.
- Private Registry / Missing ImagePullSecret: Pod pulling from third-party
  registry without valid imagePullSecrets configured in pod spec or service
  account.
- VPC Service Controls Perimeter Block: Network egress to Artifact Registry
  blocked by VPC-SC policy or lack of Private Google Access on the node subnet.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If missing tag/typo: Correct the image name or tag in the Deployment
   spec.template.spec.containers[*].image.
2. If IAM permission missing: Grant roles/artifactregistry.reader to the GKE
   node pool service account.
3. If third-party registry: Create a Kubernetes secret with docker-registry
   credentials and reference it in imagePullSecrets.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
