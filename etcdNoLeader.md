# PagerDuty Alert ID: etcdNoLeader
# Description
This alert is triggered when ETCD does not report a leader to Prometheus metrics. When ETCD has no leader it is unable to process any writes. 
# Investigation and Triage

First check to make sure that the `etcdInsufficientMembers` or the `etcdMembersDown` alert are not firing. The `etcdNoLeader` is usually caused by one or both. If they are not firing (or `Pending`) after 5 minutes, you will need to debug this issue.

The first thing to determine is whether all the control plane are up. After logging into the cluster with `oc login` check the nodes:

```
oc get nodes

NAME                            STATUS   ROLES          AGE   VERSION
master-0.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
master-1.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
master-2.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
worker-0.lab-cluster.ocp4.lab   Ready    infra,worker   14d   v1.18.3+ca4017d
worker-1.lab-cluster.ocp4.lab   Ready    infra,worker   14d   v1.18.3+ca4017d
worker-2.lab-cluster.ocp4.lab   Ready    infra,worker   14d   v1.18.3+ca4017d
```

If all of the nodes are `Ready`, get a list of all of the ETCD pods: 

```
oc get pods -n openshift-etcd

NAME                                              READY   STATUS      RESTARTS   AGE
etcd-master-0.lab-cluster.ocp4.lab                4/4     Running     0          13d
etcd-master-1.lab-cluster.ocp4.lab                4/4     Running     0          13d
etcd-master-2.lab-cluster.ocp4.lab                4/4     Running     0          13d
revision-pruner-4-master-0.lab-cluster.ocp4.lab   0/1     Completed   0          13d
revision-pruner-4-master-1.lab-cluster.ocp4.lab   0/1     Completed   0          13d
revision-pruner-4-master-2.lab-cluster.ocp4.lab   0/1     Completed   0          13d
```

Connect to a specific pod:

```
oc rsh -n openshift-etcd -c etcdctl etcd-master-0.lab-cluster.ocp4.lab
```

Start by getting the member list:

```
etcdctl member list --write-out=table
+------------------+---------+-------------------------------+----------------------------+----------------------------+------------+
|        ID        | STATUS  |             NAME              |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |
+------------------+---------+-------------------------------+----------------------------+----------------------------+------------+
| 5c0214bbf7570840 | started | master-1.lab-cluster.ocp4.lab | https://10.120.120.51:2380 | https://10.120.120.51:2379 |      false |
| 641550c2d0273371 | started | master-0.lab-cluster.ocp4.lab | https://10.120.120.50:2380 | https://10.120.120.50:2379 |      false |
| 9c5c7bb5cbe5284c | started | master-2.lab-cluster.ocp4.lab | https://10.120.120.52:2380 | https://10.120.120.52:2379 |      false |
+------------------+---------+-------------------------------+----------------------------+----------------------------+------------+
```

If the members are online, check the endpoint status which contains information about which host is currently the leader:

```
etcdctl endpoint status --write-out=table
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.120.120.52:2379 | 9c5c7bb5cbe5284c |   3.4.9 |  171 MB |     false |      false |      3932 |    7310581 |            7310581 |        |
| https://10.120.120.50:2379 | 641550c2d0273371 |   3.4.9 |  171 MB |      true |      false |      3932 |    7310586 |            7310586 |        |
| https://10.120.120.51:2379 | 5c0214bbf7570840 |   3.4.9 |  171 MB |     false |      false |      3932 |    7310586 |            7310586 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Once you have looked at the status, if there is nothing obvious, look at the events in the project:

```
oc project openshift-etcd
oc get events --sort-by=.metadata.creationTimestamp
```

If there are not immediate errors present in the event log, start inspecting the individual pod logs:

```
oc logs -c etcdctl etcd-master-0.lab-cluster.ocp4.lab
oc logs -c etcd-metrics etcd-master-0.lab-cluster.ocp4.lab
```

Frequent leader changes or no leader elections can be a sign of insufficient resources, high network latency or slow disk access.

Log into a host and look at the live resource usage in `top`:
```
oc -n default debug node/node1

chroot /host

top
```

Also consider the amount of ram in use:

```
free -m
```

If you are still unsure how to proceed start by looking at the disk performance of the VM host. Examin the networking and the resource contention on the VM host as well. Zabbix and Grafana should provide a dashboard for these metric trends.

# Verification

Follow the steps above to log into an ETCD pod and rerun the status:

```
etcdctl endpoint status --write-out=table
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.120.120.52:2379 | 9c5c7bb5cbe5284c |   3.4.9 |  171 MB |     false |      false |      3932 |    7310581 |            7310581 |        |
| https://10.120.120.50:2379 | 641550c2d0273371 |   3.4.9 |  171 MB |      true |      false |      3932 |    7310586 |            7310586 |        |
| https://10.120.120.51:2379 | 5c0214bbf7570840 |   3.4.9 |  171 MB |     false |      false |      3932 |    7310586 |            7310586 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Once there is a leader, the alert should clear in 2-4 minutes.