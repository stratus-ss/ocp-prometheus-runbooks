# PagerDuty Alert ID: etcdMembersDown
# Description
This alert indicates that one or more of the ETCD members is offline.

#  Severity & Impact

This is a critical  situation and needs to be addressed immediately. In etcd, a majority of (n/2)+1 has to agree on membership changes in order to avoid a split-brain. 

The average cluster has 3 control plane, if a single control plane is down, there is no impact to the cluster. If more than (n/2) is down, this is a severe problem and the cluster is in a Read-Only state.

# Investigation and Triage

The first thing to determine is whether all the control plane are up. After logging into the cluster with `oc login` check the nodes:

```
oc get nodes -l node-role.kubernetes.io/master=

NAME                            STATUS   ROLES          AGE   VERSION
master-0.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
master-1.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
master-2.lab-cluster.ocp4.lab   Ready    master         14d   v1.18.3+ca4017d
```

If one or more of the control planes are `Not Ready` , start by checking for unsigned CSRs:

```
NAME                                             AGE   SIGNERNAME                                    REQUESTOR                                                                         REQUESTEDDURATION   CONDITION
csr-48xxs                                        89m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper         <none>              Pending
```

You can approve any outstanding CSRs:

```
for x in `oc get csr --no-headers |grep Pend |awk '{print $1}'`; do oc adm certificate approve $x; done
```

If all of the VMs are healthy, get a list of all of the ETCD pods: 

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

If the members are online, check the endpoint health:

```
etcdctl endpoint health --write-out=table
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://10.120.120.50:2379 |   true | 13.867431ms |       |
| https://10.120.120.51:2379 |   true | 14.265281ms |       |
| https://10.120.120.52:2379 |   true |  19.23994ms |       |
+----------------------------+--------+-------------+-------+

```

You can also check the endpoint status to see if the versions, Raft Index and Raft Applied are within normal parameters. ETCD is an 'eventually consistant' database. This means that Raft Index and Raft Applied can be expected to drift slightly. A large difference between members could indicate a network communication issue.

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

There are two valid further troubleshooting avenues. First you can attempt to kill the pods and allow OpenShift to recreate them. Do so with like this:

```
oc delete pod -n openshift-etcd etcd-master-2.lab-cluster.ocp4.lab
```

Repeat the above checks inside the ETCD pod to verify that the pod is rejoining the cluster properly.

If this does not resolve things, on a rare occassion rebooting the VM can resolve some of the sync issues.

You can try draining the nodes one at a time and rebooting them.

Drain the node:
```
oc adm drain node/node1 --ignore-daemonsets --delete-local-data
```

Log into a host and reboot it:
```
oc -n default debug node/node1

chroot /host

reboot
```

Ensure that node has come back up after a reboot:

```
oc get nodes
```

Once the node is `Ready`, make it as schedulable:

```
oc adm uncordon node/node1
```

# Verification

Use the `etcdctl member list`, `etcdctl endpoint status` and `etcdctl endpoint health` checks documented above to verify the cluster is now healthy.

Wait 3-5 minutes for the alert to clear.