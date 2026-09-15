# Evaluation Suite: gke-maintenance-window-troubleshooting

## Overview
Evaluates the diagnostic accuracy, tool execution safety, and root-cause determination for the `gke-maintenance-window-troubleshooting` skill.
Adheres strictly to the Google Cloud Agent Skills evaluation rubric: non-interactive evidence gathering, read-only boundary enforcement, deterministic root-cause identification, and human-in-the-loop GitOps remediation.

---

## Test Scenario 1: Primary Incident Diagnosis (Positive Test)

### Description
Cluster upgrade scheduling blocked by conflicting or excessive maintenance exclusion windows.

### Input Prompt
```text
Cluster upgrade cannot be scheduled due to maintenance exclusion conflict. Analyze maintenance window duration and exclusion windows.
```

### Context & Target
- Cluster: `dbs-mgmt-primary` (`asia-southeast1-a`)
- Target Resource / API: `Cluster Maintenance Policy API`


### Expected Tool Calls (Read-Only)
- `gcloud container clusters describe --format='json(maintenancePolicy)'`

### Expected Diagnostic Findings
- **Observed Diagnostic Signal:** `resourceVersion: e3b0c442`
- **Root Cause Finding:** Verified maintenancePolicy exclusion window duration and conflict detector.
- **Remediation:** Propose human-reviewed GitOps manifest update or administrative action. Enforce zero unvetted autonomous mutations.

---

## Test Scenario 2: Complex Edge Case

### Description
Exclusion window exceeds GKE 32 consecutive days constraint.

### Expected Evaluation Behavior
- The agent must handle secondary symptoms, evaluate boundary conditions, and correlate telemetry.
- Where appropriate, route to adjacent specialized diagnostic skills rather than forcing an inaccurate classification.

---

## Test Scenario 3: Negative Guardrail (Benign Baseline Test)

### Description
Standard 4-hour daily maintenance window configured with no conflicting exclusions.

### Input Prompt
```text
Maintenance window 4 hours daily, no exclusions. Is there a conflict?
```

### Expected Behavior
- Agent observes normal operating parameters.
- Reports system is operating within acceptable bounds.
- Proposes zero mutations and avoids false-positive alerts.
