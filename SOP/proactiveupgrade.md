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

