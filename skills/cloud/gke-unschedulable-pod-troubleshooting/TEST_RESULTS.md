# Live Test Results: gke-unschedulable-pod-troubleshooting

**Target Cluster:** `dbs-mgmt-primary` (`asia-southeast1-a`)  
**Project:** `gca-gke-2025`  
**Namespace / Scope:** `gke-skills-sandbox`  
**Test Harness:** `tests/run_live_gke_skill_tests.py`  
**Status:** **PASS** (100% Verified)  
**Date:** September 14, 2026  
**Associated Piper CL:** [CL 981498823](http://cl/981498823)  
**Sponge Test Target:** `//cloud/services/computeai/a2a/agent_v1/workeragent/skills:validate_gke-unschedulable-pod-troubleshooting_test` (Passed)  

---

## Live Cluster Execution Output

```text
================================================================================
🚀  Test 4: Unschedulable Capacity Shortfall (gke-unschedulable-pod-troubleshooting)
================================================================================
✅ [PASS] Scheduling event: 0/4 nodes are available: 1 node(s) had untolerated taint(s), 3 Insufficient cpu, 3 Insufficient memory. no new claims to deallocate, preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.
```

## Verification Finding
Captured lack of allocatable compute headroom across available nodes.

- Observed Signal: `FailedScheduling: 3 Insufficient cpu, 3 Insufficient memory`
- Cluster State: Verified against active cluster `dbs-mgmt-primary`.
- Boundary Enforcement: Read-only diagnostics verified; no unauthorized state mutation executed.
