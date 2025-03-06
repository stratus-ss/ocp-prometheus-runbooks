


## Description

This alert means that the 99th percentile latency of etcd's disk write-ahead log (WAL) sync operations exceeds 1 second over a 5-minute window. It’s designed to detect situations where the slowest 1% of disk syncs are taking too long.

## Severity & Impact

While this alert is critical, it indicates a slow down in ETCD and does not impact application performance. It can lead to various scenarios like leader election failure, frequent leader elections, slow reads and writes. This can eventually lead to long term degradation affecting things like new application creation and/or issues self healing.

## Diagnosis

### Logging

Logs are available in the cluster itself including any endpoint the log forwarder has been setup to accomodate. However, this problem does not usually manifest evidence in the logs.

### Metrics/Dashboard Link

The default OpenShift dashboard can be found here. 



## Past Triggers

Possible triggers for ETCD losing quorum include:

* Overworked hard disks
* Failing hard drives
* A cluster that is significantly more active than it was spec'd for
* VM or Physical host failures

> [!NOTE]
> In the event that the cluster is far busier than anticipated, the underlying cause may be something completely unrelated to ETCD
> This could be an indicator of severe application misconfiguration or a bad actor.


## Hosts


The ETCD members reside on the control plane nodes in the cluster. You can view these nodes with the following command:
```
oc get nodes --show-labels -l node-role.kubernetes.io/master
```

> [!NOTE]
> If the Fsync is sufficiently degraded the API may have trouble responding.

## Access Procedure

You may still be able to login to the cluster as normal using the oc commands. Sometimes, when the cluster is degraded enough you may have to resort to using the kubeconfigto log into the cluster. You can use the kubeconfig by exporting the path to the kubeconfig as seen below:
```
export KUBECONFIG=/path/to/kubeconfig
```
any subsequent oc commands should now be using the kubeconfig.

CAUTION: using the kubeconfig and then logging in as a regular user has the potential to pollute the kubeconfig you should start a new shell before logging in as a regular user!

With the cluster in a read only state it will not be possible to run oc debug on anything in the cluster. You will have to resort to using SSH access. This requires the SSH key that was used during cluster installation. CoreOS uses the core user so you will have to SSH similar to the below
```
ssh core@host-ip
```

## Related Automations

N/A

## Important Configurations

The official documentation should be consulted for a more indepth overview of the tuning ETCD, including having a separate disk for the database itself. You can find the ETCD configuration in the operator namespace 
```
oc edit etcd -n openshift-etcd-operator
```
Normally there is very little that you should adjust but you may wish to turn up the loglevel while attempting to debug ETCD. It may only be useful when attempting to reproduce the problem in another environment as changing the operator likely requires the a change in the cluster (which is read only if there are no leaders).

## Technical Steps (Overview)

* Ensure that the requisite pods are up
* Review the logs of the pods
* RSH into an ETCD pod and get the endpoint status
* Check the Disk I/O of the ETCD host
* Review resource allocation for ETCD hosts
* Review the network connection between hosts
* Check that ETCD is being defragged by the operator
* Check for a high number of events in the cluster

## Technical Steps

### Check Pods

You can view the status of the currently running ETCD pods with the following command:
```
oc get pods -n openshift-etcd
```
You should see pods for each ETCD member. If there are 3 member nodes (i.e. 3 control plane) you should see a pod for each.

If for some reason, you cannot use the oc commands, you can SSH to the host and run:
```
crictl ps |grep etcd
```
There are more ETCD containers running in CRIO on the host. You should expect to see:
```
etcd-operator
etcd-readyz
etcd-health-monitor
etcd-metrics
etcd
etcdctl
```

### Review Logs

If you are not centralizing your system pod logs to somewhere like Splunk, you can check the logs with the following command:

oc logs -n openshift-etcd etcd-control-plane.ocp4.stratus.lab -c etcd

You are looking for any errors or warnings in the logs. Common messages include:

* “took too long”
* “timed out”
* “slow disk”
* “slow network”
* “connection reset”
* "exceeded-duration"

### Endpoint Status 

If you can still oc rsh into a container, you can run the following commands and observe the results:
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
The endpoint status command will indicate whether or not the ETCD cluster currently has a leader.

### Check Disk I/O

Ideally, your enterprise monitoring stack will have data on how the backend storage is performing. If you want to verify this from ETCD's point of view you can use a utility called fio to help ascertain the disk performance. After you have logged into one of the control plane via either SSH or oc debug you can run the following command:
```
podman run --volume /var/lib/etcd:/var/lib/etcd:Z quay.io/openshift-scale/etcd-perf
```
This should work as it bypasses OpenShift's scheduler and runs a container directly on the host. After running this command you will see messages similar to this for a success:
```
99th percentile of fsync is 5865472 ns
99th percentile of the fsync is within the recommended threshold: - 20 ms, the disk can be used to host etcd
```
Or this for a failure:
```
99th percentile of fsync is 15865472 ns
99th percentile of the fsync is greater than the recommended value which is 20 ms, faster disks are recommended to host etcd for better performance
```

> **Warning**
>  You MUST run the above test several times to get an average and/or avoid making decisions based on an anomaly! 

Slow disks are a very common cause of ETCD failures.

In addition, etcd_disk_backend_commit_duration should be less than 25ms and etcd_disk_wal_fsync_duration should be less than 10ms. This information is available in Prometheus as well as in the OpenShift Observe dashboard:

```
histogram_quantile(0.99, sum by (instance, le) (irate(etcd_disk_wal_fsync_duration_seconds_bucket{job="etcd"}[5m])))
```

### Review Resource Allocation

If your enterprise monitoring solution is not configured to review resource utilization of the ETCD hosts you can run the following command:
```
oc adm top nodes
```
You should also ensure that your control planes are sized according to the recommended host practices documentation.



### Review the network connection

OpenShift provides a plethora of Networking related dashboards in the Observe section of the console. If the console is inaccessible, check with your centralized monitoring tools and/or networking team for more information on possible network anomalies. Some common causes of problems in ETCD regarding networking are:

* flapping interface (either at the VM/baremetal level or network switch)
* congested network switch/segment
* dropped packets
* firewall rules
* rate limiting

You can also login to each of the control plane hosts and run curl in order to test the connection time:
```
curl -k https://api.<OCP URL>.com -w "%{time_connect}\n"
```
where time_connect is expected below 2ms (0.002 in output) but usually anything above 6-8ms (0.006-0.008) means performance issues. Ping can have a different value than connection to API and if the connection time is much higher (for example 100ms) than normal latency, then there's a possibility API is not performing well and could be overcommitted.

Dropped packets or RX/TX errors can be viewed with
```
ip -s link show
ifstat <NIC>
ifstat -d 10
```

### Check ETCD Operator Defrag

Most of the time the openshift-etcd-operator in OCP version 4.10+ should automatically defrag ETCD for you. You should see defragmentation attempts in the logs of the openshift-etcd-operator pod:
```
oc logs openshift-operator-<pod id> |grep defrag
```
You can always run the defrag manually. The below command output shows you the difference between the total DB size and the DB size in use. It is recommended to run a defrag any time this gets above 40%. 
```
oc exec etcd-control-plane.ocp4.stratus.lab -c etcd -n openshift-etcd -- etcdctl endpoint status --cluster -w json | jq '.[].Status|"dbSize: " + (.dbSize|tostring) + ", dbSizeInUse: " + (.dbSizeInUse|tostring) + ", (dbSize-dbSizeInUse)/dbSize => " + ((.dbSize - .dbSizeInUse)/.dbSize*100|tostring)+"%"'
```
Before defragging the ETCD database you should run the compact it first. Log in to an ETCD pod and run the following:
```
export ETCDCTL_ENDPOINTS=$(etcdctl member list | awk -F', ' '{printf "%s%s",sep,$5; sep=","}')
```
Then get the current ETCD database revision and set it to a variable:
```
rev=$(etcdctl endpoint status --write-out="json" |  egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*' -m1)
```
Finally, issue the compact command:
```
etcdctl compact $rev
```
> **Note**
> While the compact command will decrease the size of the database, your system will not regain any space until a defrag operation has taken place.

You can run the defrag with the following commands:

> **Warning**
> The defrag operation is blocking, meaning that members will not respond until the defrag is completed

In our example, 10.120.120.51 is the leader. These IPs are the actual VM IPs so this corresponds with master-1 in this example.

Log into the ETCD pod. The commands below assume you are running them from within the ETCD pod.

Start by unsetting the ENDPOINTS:
```
unset ETCDCTL_ENDPOINTS
```
Then issue the defrag command against the localhost:
```
etcdctl --command-timeout=30s --endpoints=https://localhost:2379 defrag
```
After the defrag is completed you should see a decrease in size for the ETCD database:
```
etcdctl endpoint status -w table --cluster
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.120.120.50:2379 | 512ebba95d7a9382 |   3.4.9 |   38 MB |     false |      false |        14 |    3532784 |            3532784 |        |
| https://10.120.120.52:2379 | 693ebbb39792cfb1 |   3.4.9 |  104 MB |     false |      false |        14 |    3532784 |            3532784 |        |
| https://10.120.120.51:2379 | f9414595fddf47bd |   3.4.9 |  105 MB |      true |      false |        14 |    3532784 |            3532784 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Repeat this process for each member in the ETCD cluster ending with the member that is currently the LEADER. You should leave 2-5 minutes between defragging hosts to avoid triggering a situation where two or more members have not registered their blocking state with the cluster. This would result in issues related to quorum as you could inadvertently cause multiple hosts to be unavailable at the same time.

### Check For High Numbers Of Events

A high number of events in a cluster can cause a significant overhead in the cluster. While there is no specific threshold for what is a “high number”, generally if your cluster has over 600 events at a given time, it is considered high.  If it is possible to log into an ETCD pod run the following command:
```
etcdctl --command-timeout=60s get --prefix --keys-only / | awk -F/ '/./ { print $3 }' | sort | uniq -c | sort -n
```
Alternatively you can find the total number of events in a cluster with the following command:
```
oc get events --all-namespaces |wc -l
```
You can delete the events with the following command:
```
oc delete events --all-namespaces
```
After deleting the events, you will need to run the ETCD compact and defrag. You will also want to launch an investigation into the cause of so many events in the cluster.



### Rollback/DR Plan

You should first attempt to follow the official documentation for replacing unhealthy members. If this does not produce the desired result, it is strongly advised to open a ticket with Red Hat support and have an SME walk you through the repair of the cluster.



