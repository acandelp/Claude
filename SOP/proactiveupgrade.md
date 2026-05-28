# Proactive Upgrade SOP

## Metadata

- sop_id: proactiveUpgrade
- title: Proactive Upgrade SOP
- description: Check the cluster status before performing an upgrade

## Steps

### Step 1: Cluster Version and Upgrade Path

- Current cluster version
- Available update channels
- Any upgrade in progress (check conditions)

```text
oc get clusterversion -o yaml
```


### Step 2: Node Health

- All nodes are in "Ready" status
- No nodes in "NotReady" or "SchedulingDisabled"
- Count and identify roles (master, worker, infra)

```text
oc get nodes
```

### Step 3: Cluster Operators (CRITICAL)

- All operators show "Available=True"
- No operators are "Degraded=True" - **BLOCKING: Any degraded operator means DO NOT upgrade**
- No operators stuck in "Progressing=True" - **BLOCKING: Any progressing operator means DO NOT upgrade**

```text
oc get co
```

### Step 4: Non-Cluster Operators (OLM)

- All ClusterServiceVersions in "Succeeded" phase
- No operators in "Failed" or "Unknown" state
- List any operators that need attention

```text
oc get csv -A
```

### Step 5: Machine Config Pools

- MACHINECOUNT equals READYMACHINECOUNT
- UPDATEDMACHINECOUNT equals MACHINECOUNT
- No pools in degraded state

```text
oc get mcp
```

### Step 6: API Deprecation Check (Critical)

- APIs scheduled for removal in target version
- Usage count for each deprecated API
- **IMPORTANT**: Only flag APIs with requestCount > 0 as concerns
- **APIs with requestCount = 0**: List for information only, these are NOT a concern (API exists but not being used)
- **APIs with requestCount > 0**: FLAG as critical - workloads actively using these APIs need migration

```text
oc get apirequestcounts -o jsonpath='{range .items[?(@.status.removedInRelease!="")]}{.status.removedInRelease}{"\t"}{.status.requestCount}{"\t"}{.metadata.name}{"\n"}{end}'
```

### Step 7: Pod Disruption Budgets

- PDBs with maxUnavailable=0
- PDBs where minAvailable equals total pod count
- **IMPORTANT**: PDBs in `openshift-*` namespaces are platform-managed and should be listed for information only - they are NOT a concern
- **ONLY flag as issues**: Restrictive PDBs in customer/application namespaces (non-openshift namespaces)
- Customer PDBs can block node upgrades and should be reviewed

```text
oc get pdb -A
```

### Step 8: Persistent Volumes

- All PVs and PVCs are "Bound"
- No volumes stuck in "Terminating"
- No unmounted volumes

```text
oc get pv,pvc -A | grep -v Bound
```

### Step 9: Pod Health

- Pods not in healthy state
- CrashLoopBackOff, Error, or Pending pods
- These should be investigated before upgrade

```text
oc get pods -A | egrep -v 'Running|Completed|Succeeded'
```

### Step 10: Warning Events

- Recent cluster warnings
- Recurring issues that may affect upgrade

```text
oc get events -A --field-selector type=Warning | head -20
```
