---
name: gke-network-policy-troubleshooting
description: >-
  Diagnoses GKE inter-pod and pod-to-external communication drops caused by Kubernetes NetworkPolicies or Dataplane V2 security rules. Use when pods experience connection timeouts, refused connections, or unexpected packet drops to other cluster services. Don't use for Kubernetes Service routing and DNS failures (use gke-service-routing-troubleshooting).
---

# GKE Network Policy Troubleshooting Skill

## Purpose & Scope
This skill diagnoses networking failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by NetworkPolicy drop /
Connection timed out / Dataplane V2 packet drop.

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
Observed Symptom: Pods can resolve DNS but cannot establish TCP connections to
other pods or external IPs. Traffic is silently dropped.

Execute read-only diagnostic commands:
```bash
# 1. List all NetworkPolicies in the source and target namespaces
kubectl get networkpolicies -n {namespace} -o wide

# 2. Inspect specific NetworkPolicy YAML definitions
kubectl get networkpolicy {policy_name} -n {namespace} -o yaml

# 3. Check labels on source and destination pods
kubectl get pods -n {namespace} --show-labels

# 4. Check Dataplane V2 drop metrics and Cilium monitor logs (if Dataplane V2)
kubectl -n kube-system logs -l k8s-app=cilium --tail=50 | grep -i drop || true
```

---

### Step 2: Diagnostic Decision Tree
- Default Deny Active Without Ingress Allow: Namespace has a default-deny
  ingress policy and no rule selects the caller pod.
- Label Selector Mismatch: podSelector or namespaceSelector in NetworkPolicy
  does not match destination pod labels.
- Egress Policy Blocking DNS or External Access: Workload restricted by egress
  NetworkPolicy that does not allow port 53 (kube-dns) or external CIDRs.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If missing ingress rule: Add an ingress rule permitting traffic from the
   caller pod's label or namespace.
2. If egress blocked: Ensure egress policies include an allow rule for kube-
   dns on UDP/TCP port 53 in kube-system.
3. Test connectivity after policy update using an interactive ephemeral pod.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
