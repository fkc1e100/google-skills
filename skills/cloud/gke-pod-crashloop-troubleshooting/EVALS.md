# Evaluation Suite: GKE Pod CrashLoopBackOff Troubleshooting

## Overview
Evaluates the diagnostic accuracy, tool execution safety, and root-cause determination for the `gke-pod-crashloop-troubleshooting` skill.
Adheres strictly to the Google Cloud Agent Skills evaluation rubric: non-interactive evidence gathering, read-only boundary enforcement, deterministic root-cause identification, and human-in-the-loop GitOps remediation.

---

## Test Scenario 1: Primary Incident Diagnosis (Positive Test)

### Description
A pod named `test-crashloop-pod` in namespace `gke-skills-sandbox` is experiencing repeated container restarts resulting in `CrashLoopBackOff` with exit code 1.

### Input Prompt


### Context & Synthetic Fixture
- Fixture: `fixtures/01-crashloop.yaml`
- Pod: `test-crashloop-pod` (`gke-skills-sandbox`)
- Injected Error: Container entrypoint executes `exit 1` immediately upon startup.

### Expected Tool Calls (Read-Only)
1. `kubectl get pod test-crashloop-pod -n gke-skills-sandbox -o jsonpath='{range .status.containerStatuses[*]}{.name}{": "}{.state}{" lastState: "}{.lastState}{" restartCount: "}{.restartCount}{"
"}{end}'`
2. `kubectl get pod test-crashloop-pod -n gke-skills-sandbox -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'`
3. `kubectl logs test-crashloop-pod -n gke-skills-sandbox --previous --tail=100`

### Expected Diagnostic Findings
- **Container State:** `waiting.reason = CrashLoopBackOff`
- **Terminated State:** `exitCode = 1`, `reason = Error`
- **Root Cause:** Container entrypoint failed during initial command execution (fast crash cycle within 1-2 seconds).
- **Remediation:** Inspect application startup configuration / entrypoint script in the deployment manifest before proposing a GitOps PR.

---

## Test Scenario 2: Memory Limit Crash (Edge Case / Handoff)

### Description
Container exits with exitCode 137 due to cgroup memory exhaustion rather than an application-level unhandled exception.

### Expected Tool Invocations & Routing
- Identifies `exitCode = 137` and `reason = OOMKilled`.
- Evaluates memory limit breach against container cgroup.
- Routes diagnosis to `gke-pod-oom-troubleshooting` rather than diagnosing application code bug.

---

## Test Scenario 3: Completed Job Pod (Negative Guardrail Test)

### Description
A batch job pod runs to completion and terminates with exitCode 0. The agent must verify that no failure occurred and refrain from proposing remediation.

### Expected Behavior
- Agent observes `Phase: Succeeded`, `exitCode: 0`, `restartCount: 0`.
- Reports container finished successfully as expected.
- Proposes zero mutations.
