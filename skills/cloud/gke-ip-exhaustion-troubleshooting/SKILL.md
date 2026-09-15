---
name: gke-ip-exhaustion-troubleshooting
description: >-
  Diagnoses GKE IP address exhaustion across VPC node subnets, pod secondary CIDR ranges, and service CIDR ranges. Use when cluster operations or node pool creation fail with INVALID_IP_ADDRESS_RANGE_PATTERN or pods fail to obtain IP addresses. Don't use for network security policy blocks (use gke-network-policy-troubleshooting).
---

# GKE IP Address Exhaustion Troubleshooting Skill

## Purpose & Scope
This skill diagnoses networking failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by
INVALID_IP_ADDRESS_RANGE_PATTERN / Subnet has insufficient free IP addresses.

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
Observed Symptom: Node pool creation fails due to IP exhaustion, or pods fail
with failed to allocate for range: no IP addresses available in network.

Execute read-only diagnostic commands:
```bash
# 1. Describe GKE cluster network configuration and secondary CIDR ranges
gcloud container clusters describe "{cluster_name}" \
  --region="{location}" --project="{project_id}" \
  --format="yaml(networkConfig,ipAllocationPolicy)"

# 2. Describe VPC subnet and secondary range utilization
gcloud compute networks subnets describe "{subnet_name}" \
  --region="{region}" --project="{project_id}" \
  --format="yaml(ipCidrRange,secondaryIpRanges)"

# 3. Check node count and max pods per node settings
gcloud container node-pools list \
  --cluster="{cluster_name}" --region="{location}" --project="{project_id}" \
  --format="table(name,initialNodeCount,maxPodsConstraint.maxPodsPerNode)"

# 4. Check pod IP allocation events in the cluster
kubectl get events -A --field-selector reason=FailedCreatePodSandBox \
  --sort-by='.metadata.creationTimestamp'
```

---

### Step 2: Diagnostic Decision Tree
- Primary Subnet Exhaustion: No free IP addresses in node subnet to allocate
  to new GCE VM instances.
- Pod Secondary CIDR Exhaustion: Secondary CIDR allocated for pods is
  saturated because too many nodes are provisioned with large CIDR blocks (/24
  per node).
- Service Range Saturation: Service secondary range (/20) has exhausted
  available ClusterIPs.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If node subnet exhausted: Expand primary subnet CIDR via gcloud compute
   networks subnets expand-ip-range.
2. If pod CIDR exhausted: Add additional pod CIDR ranges using GKE multi-pod
   CIDR feature.
3. For new node pools: Reduce max-pods-per-node (e.g. from 110 to 32) to
   allocate smaller /26 CIDR blocks per node.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
