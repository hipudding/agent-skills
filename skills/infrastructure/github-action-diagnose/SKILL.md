---
name: gha-ascend-infra-detect
description: Automated diagnostic agent for GitHub Action failures on Ascend NPU infrastructure. Analyzes GHA logs, searches repository for similar issues, and performs deep K8s/Node-level troubleshooting for self-hosted runners (ARC). Use when a GHA job fails on Ascend hardware and infrastructure issues (Saturation, Hardware fault, K8s scheduling) are suspected.
---

# GHA Ascend Infrastructure Detective

This skill embodies a systematic approach to diagnosing CI failures in specialized hardware environments (Ascend NPU) using SRE principles like the USE Method (Utilization, Saturation, Errors).

## Core Diagnostic Workflow

### 1. Initial GHA Log Analysis
Analyze the failed GitHub Action log to differentiate between code bugs and infrastructure issues.
- **Signs of Code Bug**: Python `Traceback`, `AssertionError`, `FileNotFoundError` (within workspace).
- **Signs of Infra Issue**: `Runner connection lost`, `Process completed with exit code 255/137`, `Context canceled`, `Failed to initialize NPU`.

**Action**: Use `gh issue list --search "[error message]"` to check if this is a known intermittent issue or environmental flake.

### 2. K8s Level (ARC) Diagnostics
If infra is suspected, check the status of the `EphemeralRunner` and its underlying Pod.
- **Pod Scheduling**: Look for `Pending` state. Use `kubectl describe pod <pod-name>` to check for:
    - `Insufficient huawei.com/ascend-910`: Resource saturation.
    - `0/N nodes are available: ... pod has unbound immediate PersistentVolumeClaims`: Storage issue.
- **Pod Lifecycle**: Check for `OOMKilled` (137) or `Evicted`.

### 3. Node & Hardware Deep Dive (Ascend NPU)
When cluster-level status is healthy but jobs fail, perform hardware-level checks. **Requires User Consent for SSH/Exec.**

- **NPU Health Check**: Run `npu-smi info` on the target node.
    - **Health Status**: Anything other than "OK" indicates hardware/driver fault.
    - **Utilization/Saturation**: Check `HBM Used` and `NPU Usage`.
- **System Logs**: Search for driver/firmware signals.
    - `dmesg | grep -iE "hiai|npu"`: Look for PCIE errors, heartbeat loss, or firmware crashes.

## Actionable Principles (USE Method)

- **Utilization**: Check if NPUs are fully utilized across the cluster.
- **Saturation**: Check for "Queueing" signals (Pending pods, high load average on nodes).
- **Errors**: Check for hardware alarms, driver crashes, and communication timeouts.

## User Interaction Guidelines

- **Consent First**: Always ask before executing `kubectl exec`, `ssh`, or any command that interacts with the live environment.
- **Mode A (Autonomous)**: Aim to gather all evidence automatically but explain each step clearly to the user.
- **Final Opinion**: Synthesize all findings (GHA log + Issue search + K8s status + Node health) into a clear "Infrastructure Health Report" with a definitive conclusion (e.g., "Hardware Fault: Node-02 NPU Memory Error").

## Test Scenarios for Validation

1. **Scenario: Resource Queuing** - Pod is Pending due to NPU exhaustion. Skill should identify "Unschedulable" status and report saturation.
2. **Scenario: NPU Heartbeat Loss** - Job fails with connection lost. Skill should find "Heartbeat loss" in node dmesg and conclude hardware fault.
3. **Scenario: Known Flake** - Job fails with a specific error string found in 3 open issues. Skill should link issues and conclude "Known Intermittent Flake".
