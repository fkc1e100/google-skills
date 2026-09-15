---
name: gke-node-notready-troubleshooting
description: >-
  Diagnoses GKE cluster nodes reporting NotReady status, kubelet heartbeat timeouts, container runtime failures, or node network partitions. Use when nodes transition to NotReady or Ready=False. Don't use for application container crashes inside healthy nodes (use gke-pod-crashloop-troubleshooting).
---

# GKE Node NotReady Troubleshooting Skill

## Purpose & Scope
This skill diagnoses node & infrastructure failures in Google Kubernetes Engine
(GKE) clusters, specifically addressing incidents triggered by Node Ready=False
/ Node NotReady / Kubelet stopped posting node status.

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
Observed Symptom: One or more nodes in the GKE cluster transition to NotReady.
Pods on the affected nodes are evicted or become unreachable.

Execute read-only diagnostic commands:
```bash
# 1. Inspect conditions of NotReady nodes
kubectl get nodes --field-selector status.phase!=Running \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason,MSG:.status.conditions[-1].message

# 2. Inspect full condition history for the affected node
kubectl get node {node_name} \
  -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" Reason: "}{.reason}{" Msg: "}{.message}{"\n"}{end}'

# 3. Query Cloud Logging for serial console or kubelet crash logs
gcloud logging read \
  'resource.type="gce_instance" AND logName:"serialport1_console" AND resource.labels.instance_id="{instance_id}"' \
  --freshness=1h --limit=20 --project="{project_id}"

# 4. Check GCE instance status and maintenance operations
gcloud compute instances describe {node_name} \
  --zone="{zone}" --project="{project_id}" \
  --format="yaml(status,scheduling,lastStartTimestamp)"
```

---

### Step 2: Diagnostic Decision Tree
- Kubelet Stopped Posting Status: VM kernel lockup, out-of-memory kernel
  panic, or container runtime (containerd) hang preventing kubelet heartbeat.
- DiskPressure / PIDPressure: Root disk exhausted on node VM, preventing
  kubelet from recording pod lifecycle states.
- Spot / Preemptible VM Termination: Instance was preempted by Compute Engine.
  GKE node auto-repair will provision a replacement node.
- Network Partition: Node lost connectivity to GKE control plane (blocked port
  443 or routes deleted in VPC).

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If transient host failure: Trigger GKE node auto-repair or recreate node
   pool instance via gcloud compute instances reset.
2. If disk full: Inspect node boot disk size in node pool configuration and
   resize if workloads exceed default storage.
3. If Spot preemption: Ensure mission-critical workloads run on standard
   (non-Spot) node pools with appropriate PDBs.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
