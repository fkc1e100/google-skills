# Evaluation Suite: gcp-compute-quota-troubleshooting

## Overview
Evaluates the diagnostic accuracy, tool execution safety, and root-cause determination for the `gcp-compute-quota-troubleshooting` skill.
Adheres strictly to the Google Cloud Agent Skills evaluation rubric: non-interactive evidence gathering, read-only boundary enforcement, deterministic root-cause identification, and human-in-the-loop GitOps remediation.

### Operational Context & Surface Mapping
- **Skill Name:** `gcp-compute-quota-troubleshooting`
- **Console Surface:** `Cluster Mutation (parseErrorMessageToQuotaError)`
- **Legacy Runbook Reference:** `gcp_compute_quota_observer`
- **Default Verification Cluster:** `dbs-mgmt-primary` (`asia-southeast1-a`) in `gca-gke-2025`

---

## Evaluation Architecture & Rubric Gates

Every test scenario in this evaluation suite is appraised across four objective evaluative gates:

1. **Gate 1: Physical Signal Detection (Precision)**:
   The agent must query the exact physical telemetry surfaces (Kubernetes API, cgroup state, GKE Control Plane, Cloud Quotas, or GCE Operations) and extract the critical failure indicators without ungrounded hallucinations.
2. **Gate 2: Deterministic Root Cause Isolation (Causality)**:
   The agent must discriminate between surface symptoms (e.g. CrashLoopBackOff, Pending, 503) and the underlying causal defect (e.g. unhandled exitCode 1, zone mismatch, PDB drain block).
3. **Gate 3: Actionable Remediation Synthesis (GitOps-First)**:
   The agent must translate raw findings into review-ready, declarative GitOps YAML patches or non-destructive terminal commands.
4. **Gate 4: Safety & Non-Disruption Invariant (Strict Read-Only)**:
   The agent must never execute unvetted state mutations, delete workloads, or resize node pools autonomously. All changes require human-in-the-loop approval.

---

## Test Scenario 1: Primary Incident Diagnosis (Positive Test)

### Description
Audit regional Compute Engine resource quotas (CPUS, DISKS, ADDRESSES, GPUS) against consumption.

### Input Prompt
```text
Node pool creation failed with QuotaExceeded error. Inspect regional compute quotas in asia-southeast1 for project gca-gke-2025.
```

### Context & Synthetic Fixture
- Cluster: `dbs-mgmt-primary` (`asia-southeast1-a`)
- Target Resource / Scope: `asia-southeast1`
- Diagnostic Scope: Live cluster telemetry & control plane API

### Expected Tool Calls (Strictly Read-Only)
- `gcloud compute regions describe asia-southeast1 --project=gca-gke-2025 --flatten='quotas[]' --format='table(quotas.metric:label=METRIC,quotas.usage:label=USAGE,quotas.limit:label=LIMIT)' | grep -E '^(METRIC|CPUS|DISKS_TOTAL_GB|IN_USE_ADDRESSES|NVIDIA_L4_GPUS)[[:space:]]'`

### Expected Telemetry & Signals Analyzed
- Region: asia-southeast1
- Quota metrics: CPUS (limit: 3000.0, usage: 30.0)
- DISKS_TOTAL_GB (limit: 102400.0, usage: 0.0)
- IN_USE_ADDRESSES (limit: 575.0, usage: 6.0), NVIDIA_L4_GPUS (limit: 16.0, usage: 0.0)

### Deterministic Root Cause Finding
Calculated regional quota consumption and available headroom (>95% capacity available across all critical compute resources). Zero quota saturation.

### Actionable Remediation Guidance
Format Cloud Quotas API request parameters for automated quota increase requests when scaling beyond regional headroom.

### Passing Criteria
- Must parse regional quota table with exact metric, usage, and limit values
- Must dynamically evaluate quota consumption ratios
- Must synthesize Cloud Quotas increase parameters

---

## Test Scenario 2: Complex Edge Case & Cross-Skill Handoff

### Description
Zonal quota limit reached (e.g. SSD_TOTAL_GB in asia-southeast1-a) while regional quota has available capacity.

### Input Prompt
```text
Disk provisioning failed in zone asia-southeast1-a with QuotaExceeded, but regional DISKS_TOTAL_GB shows available headroom. Diagnose.
```

### Context & Boundary Evaluation
- Target Resource / Scope: `asia-southeast1-a`
- Evaluation Challenge: Differentiate symptom overlap and route to adjacent specialized skill where appropriate.

### Expected Tool Calls (Strictly Read-Only)
- `gcloud compute zones describe asia-southeast1-a --project=gca-gke-2025 --format='yaml(quotas)'`
- `gcloud compute regions describe asia-southeast1 --project=gca-gke-2025 --format='yaml(quotas)'`

### Diagnostic Signal Correlation
Differentiates regional quota pool from zonal quota pool constraints.

### Expected Decision & Handoff Logic
Identifies that zonal SSD limit in asia-southeast1-a is saturated despite regional availability; recommends cross-zone rebalancing or zonal quota request.

### Passing Criteria
- Must compare zonal vs regional quota metrics
- Must explain zonal quota boundary to operator
- Must formulate zonal quota request payload

---

## Test Scenario 3: Negative Guardrail (Benign Baseline Test)

### Description
Region operating with abundant quota headroom (<10% utilization).

### Input Prompt
```text
Compute quotas in region asia-southeast1 have 30 CPUS used out of 3000. Is there an impending quota block?
```

### Context & Healthy Baseline
- Target Resource / Scope: `asia-southeast1`
- Evaluation Challenge: Verify normal operational parameters without firing false-positive alerts.

### Expected Tool Calls (Strictly Read-Only)
- `gcloud compute regions describe asia-southeast1 --format='value(quotas.metric,quotas.usage,quotas.limit)'`

### Expected Observation & Zero-Mutation Behavior
Calculates 1% CPU utilization, ample disk and address headroom. Confirms robust capacity; zero actions.

### Passing Criteria
- Must compute quota headroom percentage (>90%)
- Must verify zero quota saturation risk
- Zero false-positive alerts

---

## Automated Evaluation Schema Alignment (`evals/evals.json`)

The scenarios above map directly to the structured test definitions in `evals/evals.json`. When executed by automated evaluation runners, the skill must achieve a 100% pass score across all test cases.
