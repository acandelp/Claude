# Output Examples - ETCD Overload Check

## Example 1: Cluster with overload messages

```
$ oc whoami
system:serviceaccount:openshift-backplane:backplane-cluster-admin

Checking etcd overload messages...

WARNING: Found 1247 "server is likely overloaded" messages in etcd logs

etcd-master-0: 423 messages
etcd-master-1: 401 messages
etcd-master-2: 423 messages

Top 10 logs with highest exceeded-duration:

2026-05-19T10:23:45.123Z    etcd-master-0   100ms   2.3s
2026-05-19T10:22:15.456Z    etcd-master-1   100ms   1.8s
2026-05-19T10:21:30.789Z    etcd-master-2   100ms   1.5s
2026-05-19T10:20:45.012Z    etcd-master-0   100ms   1.2s
2026-05-19T10:19:50.345Z    etcd-master-1   100ms   980.5ms
2026-05-19T10:18:25.678Z    etcd-master-0   100ms   875.3ms
2026-05-19T10:17:10.901Z    etcd-master-2   100ms   756.2ms
2026-05-19T10:16:05.234Z    etcd-master-1   100ms   645.8ms
2026-05-19T10:15:00.567Z    etcd-master-0   100ms   589.4ms
2026-05-19T10:14:55.890Z    etcd-master-2   100ms   502.1ms
```

## Example 2: Cluster without overload messages

```
$ oc whoami
system:serviceaccount:openshift-backplane:backplane-cluster-admin

Checking etcd overload messages...

No "server is likely overloaded" messages found in etcd logs.

✅ No overload messages found in etcd.
```

## Example 3: Permission error

```
$ oc whoami
developer@example.com

$ source ETCDTroubleshooting.sh && ETCD_overload

Error from server (Forbidden): pods is forbidden: User "developer@example.com" cannot list resource "pods" in API group "" in the namespace "openshift-etcd"

❌ User doesn't have permissions to read logs in openshift-etcd. 
   Needs cluster-reader permissions or higher.
```

## Example 4: No active session

```
$ oc whoami
error: You must be logged in to the server (Unauthorized)

❌ Must log in with 'oc login' before continuing.
```
