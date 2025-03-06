# PagerDuty Alert ID: NodeFileSystemSpaceFillingUp
# Description

This fires when there is less than 10% space left on the node. It then attempts to predict whether or not the host will run out of space in the next 4 hours.

## Severity and Impact

If a file system fills up and runs out of space, processes that need to write to the file system can no longer do so, which can result in lost data and system instability.


# Investigation and Triage

The investigation steps will depend on the type of node that is throwing the error. If this is a control plane host, it is possible that ETCD is taking up a significant amount of space. 

## Compact ETCD Database

In order to attempt to reduce the size of the ETCD database you can `compact` the database. After logging in to the OpenShift cluster, start by changing into the project `openshift-etcd`:

```
oc project openshift-etcd
```

Get a list of the ETCD pods:

```
oc get pods |grep -v Completed
```

Next `rsh` into one of the pods and view the `endpoint status`

```
etcdctl endpoint status -w table --cluster
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.120.120.50:2379 | 512ebba95d7a9382 |   3.4.9 |  105 MB |     false |      false |        14 |    3523158 |            3523158 |        |
| https://10.120.120.52:2379 | 693ebbb39792cfb1 |   3.4.9 |  104 MB |     false |      false |        14 |    3523158 |            3523158 |        |
| https://10.120.120.51:2379 | f9414595fddf47bd |   3.4.9 |  105 MB |      true |      false |        14 |    3523158 |            3523158 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Export the endpoints for easier reference:

```
export ETCDCTL_ENDPOINTS=$(etcdctl member list | awk -F', ' '{printf "%s%s",sep,$5; sep=","}')
```

Then get the current ETCD database revision and set it to a variable:

```
rev=$(etcdctl endpoint status --write-out="json" |  egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*' -m1)
```

Finally, issue the `compact` command:

```
etcdctl compact $rev
```

*NOTE*: The while the `compact` command will decrease the size of the database, your system will not regain any space until a defrag operation has taken place.

## ETCD Defrag

**IMPORTANT NOTE:** The Defrag should be on on the `LEADER` last. Make sure to do all the replicas first. The `defrag` operation is blocking meaning that the member will not respond until the defrag has completed.

In our example, 10.120.120.51 is the leader. These IPs are the actual VM IPs so this corresponds with master-1 in this example.

Start by unsetting the `ENDPOINTS` variable that was defined in the previous set:

```
unset ETCDCTL_ENDPOINTS
```

Then issue the `defrag` command against the localhost:

```
etcdctl --command-timeout=30s --endpoints=https://localhost:2379 defrag
```

After the `defrag` is completed you should see a decrease in size for the ETCD database:

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

Repeat this process for each member in the ETCD cluster ending with the member that is currently the `LEADER`. You should leave 2-5 minutes between defragging hosts to avoid triggering a situation where two or more members have not registered their `blocking` state with the cluster. This would result in issues related to quorum as you could inadvertantly cause multiple hosts to be unavailable at the same time.

## Clearing Up Journald

For the most part the log rotation scripts that ship with the platform keep minimal logs on the system itself. However, in the event of an emergency you can always use the `--vacuum-time` option to purge old logs. Most of the time there are only 2 days worth of logs on a given host. To purge all but the last 8 hours worth of logs from the local host follow these steps. 

First log into the host:

```
oc debug node/master-0.lab-cluster.ocp4.lab
chroot /host
```

Remove old logs:

```
journalctl --vacuum-time=8h
```

In a small cluster this can remove 2-6G worth of space. In busier clusters this could be significantly more.

This step is meant as a temporary step to help remediate the immediate problem. However, you should investigate the long term cause of the used space and determine whether or not, for the size of your cluster, you require more space to be added to the node.

## Removing Pods With Completed Status

Another method for cleaning up space is to remove any pods that have a `Completed` status. These pods are no longer needed and can be removed with the following command:

```
oc delete pod --field-selector=status.phase==Succeeded --all-namespaces
```

This may or may not free up significant space on your nodes depending on the activity in your cluster.

## Removing Unused Containers on OCP4 Nodes

Normally, Red Hat does not recommend the use of SSH, however for this step, since we will be dealing directly with cleanup of containers on a specific host, SSH may be the best option for connectivity.

```
ssh core@master-0.lab-cluster.ocp4.lab
```

Check the space available on the device:

```
sudo df -h /var/lib/containers/storage/
```

Use Podman to clean the system:

```
sudo podman system prune -a

WARNING! This will remove:
        - all stopped containers
        - all stopped pods
        - all dangling images
        - all build cache
Are you sure you want to continue? [y/N] y
Deleted Pods
Deleted Containers
Deleted Images
```


# Verification

Verify that the hosts have enough space now:

```
df - /
Filesystem                            Size  Used Avail Use% Mounted on
/dev/mapper/coreos-luks-root-nocrypt   70G  9.7G   60G  14% /

```

*NOTE:* the `/` and `/var` are on the same device mapper and therefor share the same space. Therefore you should only have to verify with one or the other mount points.

Wait 5-15 minutes for the alert to clear.