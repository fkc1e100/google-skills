---
name: gke-service-routing-troubleshooting
description: >-
  Diagnoses GKE Service discovery and traffic routing issues, including empty endpoints, missing EndpointSlices, failing readiness probes, or misconfigured target ports. Use when requests to ClusterIP, NodePort, or LoadBalancer services fail or return connection refused. Don't use for NetworkPolicy drops (use gke-network-policy-troubleshooting).
---

# GKE Service Routing & Endpoints Troubleshooting Skill

## Purpose & Scope
This skill diagnoses networking failures in Google Kubernetes Engine (GKE)
clusters, specifically addressing incidents triggered by Service connection
refused / Empty endpoints / EndpointSlice 0 endpoints.

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
Observed Symptom: Requests to Kubernetes Service fail with connection refused or
timeout. Service has 0 endpoints.

Execute read-only diagnostic commands:
```bash
# 1. Describe Service and verify port mappings and selector labels
kubectl get service {service_name} -n {namespace} -o yaml

# 2. Inspect EndpointSlices associated with the service
kubectl get endpointslices -l kubernetes.io/service-name={service_name} \
  -n {namespace} -o yaml

# 3. Verify matching pods and their readiness status
kubectl get pods -l {selector_labels} -n {namespace} -o wide

# 4. Check readiness probe failures on backend pods
kubectl get events -n {namespace} --field-selector reason=Unhealthy \
  --sort-by='.metadata.creationTimestamp'
```

---

### Step 2: Diagnostic Decision Tree
- Selector Label Mismatch: Service spec.selector does not match any running
  pods in the namespace.
- Backend Pods Not Ready: Pods match selector but failing readiness probes
  prevent them from being registered as endpoints.
- TargetPort Mismatch: Service spec.ports[*].targetPort does not match the
  actual port opened by the application container.
- CoreDNS Failure: Cluster DNS cannot resolve
  {service_name}.{namespace}.svc.cluster.local.

---

### Step 3: Root-Cause Synthesis & Remediation Plan
Once the root cause is isolated, propose concrete remediation steps:
1. If selector mismatch: Update Service spec.selector to match pod template
   labels.
2. If readiness probe failing: Fix backend health check endpoint or adjust
   probe parameters.
3. If targetPort mismatch: Align targetPort in Service YAML with
   containerPort in Deployment spec.

### Operational Guarantees
- Read-Only Verification: All diagnostic queries use read-only inspection commands.
- Human Approval: State changes must flow through review-ready GitOps pull requests or explicit administrator action.
