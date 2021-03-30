# PagerDuty Alert ID: KubePersistentVolumeErrors
# Description
This fires when a persistent volume in the projects (openshift-.*|kube-.*|default|logging) are in either Pending or Failed state.

# Investigation and Triage

These alerts are cuased by either the persistent storage backend being full, communication issue with network storage or an underlying storage problem. Below is an overview of the OpenShift troubleshooting steps.

The first thing to determine is what errors you are receiving in the logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the application pods in question. You will need to change to the project in which the pods reside.

The AlertManager alerts should give you an indication of which project might be suffering the problem. At very least, you should have the name of the persistent volume. 

You can get the list of the persistent volumes in the cluster with the following command:

```
oc get pv
```

Volumes are stored independant of the project with which they may be associated.  Here is an example:

```
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                               STORAGECLASS   REASON    AGE
apt-cacher-vol            10Gi       RWO            Retain           Bound       repos/apt-cacher-claim                                       1y
next-certs                2Gi        RWX            Retain           Bound       nextcloud/next-cert-claim                                    202d
nextcloud-data-vol        10Gi       RWO            Retain           Bound       nextcloud/nextcloud-data-claim                               202d
nextcloud-db-volume       10Gi       RWO            Retain           Bound       nextcloud/mariadb                                            202d
nextcloud-www-vol1        10Gi       RWX            Retain           Bound       nextcloud/nextcloud-www-claim                                202d
smokeping-cache-volume    10Gi       RWO            Retain           Bound       monitoring/smokeping-cache-claim                             1y
smokeping-data-volume     10Gi       RWO            Retain           Bound       monitoring/smokeping-data-claim                              1y
unifi-controller-config   10Gi       RWO            Retain           Bound       unifi/unifi-config-claim                                     297d
wikijs-postgresql-data    10Gi       RWO            Retain           Bound       wikijs/wikijs-postgresql-pvc                                 305d
registry311-volume        20Gi       RWX            Retain           Bound       default/registry311-claim                                    1y
```

From this output you can determine the project in question as the claim lists them in `<claim>/<claim-name>` format. Once you have determined the claim and project you can proceed to troubleshoot.

If for example, the issue was with the registry change to the project, in this case `default` and examin the events in the project

```
oc project default
oc get events --sort-by=.metadata.creationTimestamp
```

You can also `rsh` into the pod which has the volume and check to see if it is full:

```
oc rsh docker-registry-7-rgzr2
```

Start with the `df` commands. You should check for free space as well as inodes free. Usually, there are alerts for both space and inode usage for PVs in the (openshift-.*|kube-.*|default|logging) projects which would indicate the problem. However, sometimes these alerts are missed or do not fire as expected. Use the following commands to verifiy (output has been truncated):

```
df -i
Filesystem                                            Inodes  IUsed      IFree IUse% Mounted on
overlay                                             26701824 133463   26568361    1% /
192.168.1.15:/storage/vms/origin_nfs/registry311 7965326118 787782 7964538336    1% /registry
/dev/vdb                                            10485760    463   10485297    1% /etc/hosts


df -h
Filesystem                                         Size  Used Avail Use% Mounted on
overlay                                             26G  7.2G   19G  29% /
192.168.1.15:/storage/vms/origin_nfs/registry311  4.2T  480G  3.8T  12% /registry
/dev/vdb                                            20G  1.5G   19G   8% /etc/hosts
```

In this case, the storage seems fine from an OpenShift perspective. As a last resort you can attempt to kill the pod or redeploy it to see if forceably remounting the PV will correct the problems:

```
oc delete pod docker-registry-7-rgzr2
```

If this does not resolve the alert after 15 minutes, you will have to continue troubleshooting network and storage provider issues which is outside the scope of this runbook.

# Verification

Verify that the pods have been recreated successfully:

```
oc get pods
```

Ensure that there are no errors present in the event logs:

```
oc get events --sort-by=.metadata.creationTimestamp
```

Verify that the alert is no longer firing in AlertManager.