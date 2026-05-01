---
name: proactiveupgrade
description: Comprehensive pre-upgrade health checks for ROSA/OSD clusters following Red Hat best practices
disable-model-invocation: true
allowed-tools: Bash(oc get nodes*), Bash(oc get clusterversion*), Bash(oc get co*), Bash(oc get csv*), Bash(oc get pdb*), Bash(oc get mcp*), Bash(oc get pv*), Bash(oc get pvc*), Bash(oc get pods*), Bash(oc get apirequestcounts*), Bash(oc get events*)
elevate-permissions: false
---

# Pre-Upgrade Health Checks

Before starting, read EXAMPLES.md in this skill directory to see the expected output format.

This skill performs comprehensive pre-upgrade validation based on:
- Red Hat Article 6955985 (API Deprecation Assessment)
- Red Hat Article 7004992 (Pre-Upgrade Requirements)
- Red Hat Article 6989139 (PROACTIVE Case Best Practices)

Execute the following checks and provide outputs only (no actions):

## 1. Cluster Version and Upgrade Path
Run `oc get clusterversion -o yaml` to check:
- Current cluster version
- Available update channels
- Any upgrade in progress (check conditions)

## 2. Node Health
Run `oc get nodes` to verify:
- All nodes are in "Ready" status
- No nodes in "NotReady" or "SchedulingDisabled"
- Count and identify roles (master, worker, infra)

## 3. Cluster Operators (CRITICAL)
Run `oc get co` to check:
- All operators show "Available=True"
- No operators are "Degraded=True" - **BLOCKING: Any degraded operator means DO NOT upgrade**
- No operators stuck in "Progressing=True" - **BLOCKING: Any progressing operator means DO NOT upgrade**

## 4. Machine Config Pools
Run `oc get mcp` to verify:
- MACHINECOUNT equals READYMACHINECOUNT
- UPDATEDMACHINECOUNT equals MACHINECOUNT
- No pools in degraded state

## 5. Non-Cluster Operators (OLM)
Run `oc get csv -A` to check:
- All ClusterServiceVersions in "Succeeded" phase
- No operators in "Failed" or "Unknown" state
- List any operators that need attention

## 6. API Deprecation Check (Critical)
Run `oc get apirequestcounts -o jsonpath='{range .items[?(@.status.removedInRelease!="")]}{.status.removedInRelease}{"\t"}{.status.requestCount}{"\t"}{.metadata.name}{"\n"}{end}'` to identify:
- APIs scheduled for removal in target version
- Usage count for each deprecated API
- **IMPORTANT**: Only flag APIs with requestCount > 0 as concerns
- **APIs with requestCount = 0**: List for information only, these are NOT a concern (API exists but not being used)
- **APIs with requestCount > 0**: FLAG as critical - workloads actively using these APIs need migration

## 7. Pod Disruption Budgets
Run `oc get pdb -A` to check for restrictive PDBs:
- PDBs with maxUnavailable=0
- PDBs where minAvailable equals total pod count
- **IMPORTANT**: PDBs in `openshift-*` namespaces are platform-managed and should be listed for information only - they are NOT a concern
- **ONLY flag as issues**: Restrictive PDBs in customer/application namespaces (non-openshift namespaces)
- Customer PDBs can block node upgrades and should be reviewed

## 8. Persistent Volumes
Run `oc get pv,pvc -A | grep -v Bound` to verify:
- All PVs and PVCs are "Bound"
- No volumes stuck in "Terminating"
- No unmounted volumes

## 9. Pod Health
Run `oc get pods -A | egrep -v 'Running|Completed|Succeeded'` to identify:
- Pods not in healthy state
- CrashLoopBackOff, Error, or Pending pods
- These should be investigated before upgrade

## 10. Warning Events
Run `oc get events -A --field-selector type=Warning | head -20` to check:
- Recent cluster warnings
- Recurring issues that may affect upgrade

**Output Format:**
Present findings in this structure:

```
=== CLUSTER VERSION ===
Current: X.Y.Z
Channel: [stable/candidate/fast]-X.Y

=== NODE HEALTH ===
Total: X nodes
Ready: X | NotReady: X
Roles: X masters, X workers, X infra

=== CLUSTER OPERATORS ===
Total: X
Available: X | Degraded: X | Progressing: X
[List any degraded operators]

=== MACHINE CONFIG POOLS ===
[Pool name]: Ready X/X, Updated X/X, Degraded: [true/false]

=== NON-CLUSTER OPERATORS ===
Total: X
Succeeded: X | Failed: X
[List any failed CSVs]

=== API DEPRECATIONS ⚠️ ===
Deprecated APIs with active usage (requestCount > 0): X
Deprecated APIs with no usage (requestCount = 0): X

⚠️ ACTIVE USAGE (ACTION REQUIRED):
[Version] | [Count] | [API Name]
[Only list APIs with requestCount > 0 - these need migration]

Informational (no action needed):
[Version] | [Count: 0] | [API Name]
[List APIs with requestCount = 0 - not a concern]

If no deprecated APIs found at all: "None found"

=== POD DISRUPTION BUDGETS ===
Total PDBs: X
OpenShift-managed PDBs: X (informational - not a concern)
Customer PDBs with restrictions: X

[List openshift-* PDBs for information only]
[Only flag customer namespace PDBs as issues if restrictive]

=== PERSISTENT VOLUMES ===
Unbound PVs: X
Stuck PVCs: X
[List any issues]

=== UNHEALTHY PODS ===
Total unhealthy: X
[List pods not in Running/Completed/Succeeded]

=== RECENT WARNINGS ===
[Top 5-10 warning events if significant]

=== RECOMMENDATION ===
✅ Cluster is ready for upgrade
OR
⚠️ Issues detected - resolve before upgrading:
1. [Issue 1]
2. [Issue 2]
```

**CRITICAL UPGRADE BLOCKING CONDITIONS:**
- ANY Cluster Operator showing Degraded=True → ❌ DO NOT UPGRADE
- ANY Cluster Operator showing Progressing=True → ❌ DO NOT UPGRADE
- ANY Node in NotReady state → ❌ DO NOT UPGRADE
- ANY Machine Config Pool degraded → ❌ DO NOT UPGRADE
- Deprecated APIs with active usage (requestCount > 0) → ⚠️ REVIEW CAREFULLY - workloads must migrate
  - **Note**: Deprecated APIs with requestCount = 0 are NOT a concern (API exists but not being used)
- ANY Non-Cluster Operator (CSV) in Failed state → ⚠️ REVIEW BEFORE UPGRADE

**Advisory (investigate but may not block):**
- Restrictive Customer PDBs in non-openshift namespaces (may cause upgrade delays)
  - **Note**: PDBs in openshift-* namespaces are platform-managed and NOT a concern
- Unbound PVs/PVCs
- Unhealthy pods (investigate root cause)
- Warning events (review for patterns)

Provide clear, actionable outputs so CRE can assess upgrade readiness.
