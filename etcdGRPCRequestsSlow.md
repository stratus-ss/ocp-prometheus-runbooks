# PagerDuty Alert ID: etcdGRPCRequestsSlow
# Description
Etcd uses gRPC to communicate between each of the nodes in the cluster. It is good to track the performance and errors of these metrics. This alert fires if at least 1% the gRPC requests in the ETCD cluster are taking longer than 0.15 seconds as seen in a 5 minute time frame
# Investigation and Triage
This alert can be caused by multiple sources: high load, disk activity, network latency etc. As a result you will have to spend time narrowing down the source of the problem and it is not possible to document the possible solutions. Therefore what follows below are troubleshooting steps to help to get you in the right direction.

## Checking Prometheus for GRPC stats

First check for failed GRPC Requests could be checked as per following:

```
100 * sum(rate(grpc_server_handled_total{job=~".*etcd.*", grpc_code!="OK"}[5m])) BY (job, instance, grpc_service, grpc_method)
```

Network round trip time could be checked as per following, respectively:
```
histogram_quantile(0.99, rate(etcd_grpc_unary_requests_duration_seconds_bucket[5m])))
histogram_quantile(0.99,etcd_network_peer_round_trip_time_seconds_bucket)
```

## Check ETCD logs

The first thing to determine is what errors you are receiving in the ETCD logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the Prometheus pods. Change to the `openshift-etcd` project:

```
oc project openshift-etcd
```

List the pods to get the names of the pods you want to get logs from:

```
oc get pods

NAME                                              READY   STATUS      RESTARTS   AGE
etcd-master-0.lab-cluster.ocp4.lab                4/4     Running     0          4d20h
etcd-master-1.lab-cluster.ocp4.lab                4/4     Running     0          4d20h
etcd-master-2.lab-cluster.ocp4.lab                4/4     Running     1          4d20h
installer-2-master-0.lab-cluster.ocp4.lab         0/1     Completed   0          4d20h
installer-2-master-1.lab-cluster.ocp4.lab         0/1     Completed   0          4d20h
installer-2-master-2.lab-cluster.ocp4.lab         0/1     Completed   0          4d20h
revision-pruner-2-master-0.lab-cluster.ocp4.lab   0/1     Completed   0          4d20h
revision-pruner-2-master-1.lab-cluster.ocp4.lab   0/1     Completed   0          4d20h
revision-pruner-2-master-2.lab-cluster.ocp4.lab   0/1     Completed   0          4d20h
```

View the logs from a specific pod:

```
oc logs etcd-master-0.lab-cluster.ocp4.lab -c etcd | less
```

Some examples of messages indicating latency are the following:
```
W | etcdserver: apply entries took too long [$x ms for 1 entries]
W | etcdserver: avoid queries with large range/delete range!
```

Alternatively, if the `oc` commands are taking a long time to respond, you could dump the logs directly from the control plane. This method may also expose additional symptoms that might not be visable from the pod logs themselves. The following command will create log files in `/tmp/` on the host the command is being run from:
```
for node in `oc get nodes --selector='node-role.kubernetes.io/master' -oname | awk -F/ '{print $2}' ` ; 
    do echo $node ; 
    oc debug node/$node -- chroot /host /bin/bash -c ' /usr/bin/crictl logs `crictl ps --label io.kubernetes.container.name=etcd -q` ' &>/tmp/$node.etcd.log ; 
done
```


## Check the ETCD Health
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

### Check Prometheus Metrics

There are two metrics that should be checked. To rule out a slow disk, monitor `backend_commit_duration_seconds` (p99 duration should be less than 25ms). The Prometheus query for this is:

```
histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))
```

The `wal_fsync_duration_seconds` (p99 duration should be less than 10ms), to confirm the disk is reasonably fast. You can check this with this query:

```
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))
```

If the disk is too slow, assigning a dedicated disk to Etcd or using a faster disk will typically solve the problem.


# Verification

The verification that this issue has been resolved is that the alert is no longer firing.