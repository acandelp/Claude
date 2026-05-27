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
