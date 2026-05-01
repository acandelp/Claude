# Expected Output Examples

## Example 1: Healthy Cluster Ready for Upgrade

```
=== CLUSTER VERSION ===
Current: 4.15.3
Channel: stable-4.15
Upgrade in progress: No

=== NODE HEALTH ===
Total: 12 nodes
Ready: 12 | NotReady: 0
Roles: 3 masters, 6 workers, 3 infra

=== CLUSTER OPERATORS ===
Total: 32
Available: 32 | Degraded: 0 | Progressing: 0

=== MACHINE CONFIG POOLS ===
master: Ready 3/3, Updated 3/3, Degraded: false
worker: Ready 6/6, Updated 6/6, Degraded: false
infra: Ready 3/3, Updated 3/3, Degraded: false

=== NON-CLUSTER OPERATORS ===
Total: 8
Succeeded: 8 | Failed: 0

=== API DEPRECATIONS ⚠️ ===
None found

=== POD DISRUPTION BUDGETS ===
Total PDBs: 8
OpenShift-managed PDBs: 8 (informational - not a concern)
Customer PDBs with restrictions: 0

OpenShift PDBs (platform-managed, no action needed):
- openshift-console/console: maxUnavailable=1
- openshift-monitoring/prometheus-k8s: minAvailable=1
- openshift-monitoring/alertmanager-main: maxUnavailable=1
- (5 more openshift-* PDBs...)

=== PERSISTENT VOLUMES ===
Unbound PVs: 0
Stuck PVCs: 0
All volumes healthy

=== UNHEALTHY PODS ===
Total unhealthy: 0
All pods in Running/Completed state

=== RECENT WARNINGS ===
No significant warnings in last 1h

=== RECOMMENDATION ===
✅ Cluster is ready for upgrade
- All nodes are healthy
- All operators are available
- No deprecated APIs in use
- No restrictive PDBs blocking upgrade
- Storage is healthy
```

## Example 2: Cluster with Critical Issues

```
=== CLUSTER VERSION ===
Current: 4.14.8
Channel: stable-4.14
Upgrade in progress: No

=== NODE HEALTH ===
Total: 9 nodes
Ready: 8 | NotReady: 1
Roles: 3 masters, 5 workers, 1 infra

NotReady nodes:
- ip-10-0-145-23.us-east-2.compute.internal (worker)

=== CLUSTER OPERATORS ===
Total: 32
Available: 31 | Degraded: 1 | Progressing: 0

Degraded operators:
- authentication: OAuthServerServiceEndpointAccessibleControllerDegraded

=== MACHINE CONFIG POOLS ===
master: Ready 3/3, Updated 3/3, Degraded: false
worker: Ready 5/6, Updated 5/6, Degraded: true
infra: Ready 1/1, Updated 1/1, Degraded: false

=== NON-CLUSTER OPERATORS ===
Total: 5
Succeeded: 4 | Failed: 1

Failed CSVs:
- openshift-custom/custom-operator.v1.2.3: InstallCheckFailed

=== API DEPRECATIONS ⚠️ ===
Deprecated APIs with active usage (requestCount > 0): 3
Deprecated APIs with no usage (requestCount = 0): 0

⚠️ ACTIVE USAGE (ACTION REQUIRED):
1.16 | 1247 | flowschemas.v1beta1.flowcontrol.apiserver.k8s.io
1.16 | 856  | prioritylevelconfigurations.v1beta1.flowcontrol.apiserver.k8s.io
1.22 | 45   | ingresses.v1beta1.networking.k8s.io

⚠️ These APIs are actively being used and workloads MUST migrate before upgrading to 4.16

=== POD DISRUPTION BUDGETS ===
Total PDBs: 10
OpenShift-managed PDBs: 8 (informational - not a concern)
Customer PDBs with restrictions: 2 ⚠️

OpenShift PDBs (platform-managed, no action needed):
- openshift-monitoring/prometheus-k8s: minAvailable=1
- openshift-console/console: maxUnavailable=1
- (6 more openshift-* PDBs...)

⚠️ CUSTOMER PDBs WITH RESTRICTIONS (ACTION REQUIRED):
- production/frontend-pdb: maxUnavailable=0 (BLOCKS node upgrades)
- production/backend-pdb: minAvailable=5, currentReplicas=5 (no disruption allowed, BLOCKS upgrades)

=== PERSISTENT VOLUMES ===
Unbound PVs: 2
Stuck PVCs: 1

Issues:
- pvc-orphaned-data (namespace: app-team) - PV Released, not Bound
- pvc-stuck-delete (namespace: ci-builds) - Terminating for 2h

=== UNHEALTHY PODS ===
Total unhealthy: 7

CrashLoopBackOff:
- openshift-custom/custom-operator-7b5d9c-xz4kp (5 restarts)
- production/backend-api-3-xk8j2 (12 restarts)

Pending:
- ci-builds/build-job-456 (Unschedulable: insufficient CPU)
- ci-builds/build-job-457 (Unschedulable: insufficient CPU)

Error:
- app-team/worker-pod-8 (Init:Error)

=== RECENT WARNINGS ===
1. Node DiskPressure detected on worker-node-2 (30m ago)
2. ImagePullBackOff for production/new-deployment (15m ago)
3. Pod evicted due to node pressure (10m ago)
4. Failed to mount volume for pvc-stuck-delete (ongoing)

=== RECOMMENDATION ===
❌ CRITICAL BLOCKING ISSUES - DO NOT UPGRADE:

**BLOCKING CONDITIONS DETECTED:**
1. ❌ Node Health: Worker node ip-10-0-145-23 is NotReady (BLOCKS UPGRADE)
2. ❌ Degraded Operator: authentication operator is Degraded (BLOCKS UPGRADE)
3. ❌ Machine Config Pool: worker pool degraded (BLOCKS UPGRADE)
4. ⚠️ Failed Operator: custom-operator installation failing (REVIEW REQUIRED)

**ADDITIONAL ISSUES TO RESOLVE:**
5. API Deprecations: 3 deprecated APIs in use - workloads must migrate before 4.16
6. Restrictive PDBs: 2 PDBs will block node upgrades in production namespace
7. Storage Issues: 2 unbound PVs and 1 stuck PVC need cleanup
8. Pod Health: 7 unhealthy pods including CrashLoopBackOff

**UPGRADE READINESS: NOT READY - MULTIPLE BLOCKING CONDITIONS**

Priority Actions (MUST resolve items 1-3 before upgrade):
1. ❌ CRITICAL: Investigate and fix NotReady worker node
2. ❌ CRITICAL: Fix authentication operator degradation
3. ❌ CRITICAL: Resolve worker pool degradation
4. Review custom-operator installation failure
5. Migrate workloads using deprecated APIs (if targeting 4.16+)
6. Adjust restrictive PDBs in production namespace
7. Clean up orphaned PVs and stuck PVCs
8. Resolve crashing pods before upgrade
```

## Example 3: Cluster with Progressing Operator (Blocked)

```
=== CLUSTER VERSION ===
Current: 4.21.4
Channel: stable-4.21
Upgrade in progress: No

=== NODE HEALTH ===
Total: 2 nodes
Ready: 2 | NotReady: 0
Roles: 2 workers (ROSA HCP)

=== CLUSTER OPERATORS ===
Total: 22
Available: 22 | Degraded: 0 | Progressing: 1

Progressing operators:
- dns: DNS "default" reports Progressing=True: "Have 1 available node-resolver pods, want 2."

=== MACHINE CONFIG POOLS ===
N/A (ROSA HCP cluster)

=== NON-CLUSTER OPERATORS ===
Total: 0
Succeeded: 0 | Failed: 0

=== API DEPRECATIONS ⚠️ ===
Deprecated APIs with active usage (requestCount > 0): 0
Deprecated APIs with no usage (requestCount = 0): 1

✅ NO ACTIVE USAGE - No action required

Informational (no action needed):
1.32 | 0 | flowschemas.v1beta3.flowcontrol.apiserver.k8s.io

Note: API is deprecated but not being used by any workloads. No migration needed.

=== POD DISRUPTION BUDGETS ===
Total PDBs: 12
OpenShift-managed PDBs: 12 (informational - not a concern)
Customer PDBs with restrictions: 0

OpenShift PDBs (platform-managed, no action needed):
- openshift-monitoring/prometheus-k8s: minAvailable=1
- openshift-console/console: maxUnavailable=1
- openshift-monitoring/alertmanager-main: maxUnavailable=1
- openshift-operator-lifecycle-manager/packageserver-pdb: maxUnavailable=1, currently 0 allowed disruptions
- (8 more openshift-* PDBs...)

Note: packageserver-pdb currently has 0 allowed disruptions but this is normal for openshift-managed PDBs

=== PERSISTENT VOLUMES ===
Unbound PVs: 0
Stuck PVCs: 0
All volumes healthy

=== UNHEALTHY PODS ===
Total unhealthy: 1
- openshift-dns/node-resolver-fndx8 (ContainerCreating for 42h)

=== RECENT WARNINGS ===
No significant warnings

=== RECOMMENDATION ===
❌ CRITICAL BLOCKING ISSUE - DO NOT UPGRADE:

**BLOCKING CONDITIONS DETECTED:**
1. ❌ Progressing Operator: DNS operator is Progressing (BLOCKS UPGRADE)
   - Missing node-resolver pod (1/2 available)
   - Pod stuck in ContainerCreating for 42h

**UPGRADE READINESS: NOT READY - OPERATOR PROGRESSING**

**WHY THIS BLOCKS UPGRADE:**
Cluster operators must be stable (Available=True, Degraded=False, Progressing=False) 
before upgrade. A progressing operator indicates an incomplete configuration change 
or ongoing issue that could interfere with the upgrade process.

Priority Actions:
1. ❌ CRITICAL: Investigate why node-resolver pod is stuck in ContainerCreating
2. ❌ CRITICAL: Resolve DNS operator progressing state
3. Verify DNS operator shows Progressing=False before proceeding with upgrade

**DO NOT PROCEED WITH UPGRADE UNTIL DNS OPERATOR IS STABLE**
```

