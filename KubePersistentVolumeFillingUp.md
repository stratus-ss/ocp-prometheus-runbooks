# PagerDuty Alert ID KubePersistentVolumeFillingUp (Critical)
# Description

This alert fires when the volume (in bytes) for a PV in (openshift-.*|kube-.*|default|logging) has less than 3% available

# Investigation and Triage

This alert is fairly straight forward and indicates that the indicated PV is nearing capacity. Generally speaking, there is no troubleshooting that needs to be done other than noting which pod is using the PV and then adjusting for more space.

If this alert has quickly followed the 15% warning version of the same VolumeFillingUp alert, this could be an indication that the application in question has either had its' storage underestimated or is using space unusually quickly.

The first thing to determine is what errors you are receiving in the logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the application pods in question. You will need to change to the project in which the pods reside.

The AlertManager alerts should give you an indication of which project might be suffering the problem. At very least, you should have the name of the persistent volume. 

You can get the list of the persistent volumes in the cluster with the following command:

```
oc get pv
```

From this output you can determine the project in question as the claim lists them in `<claim>/<claim-name>` format. Once you have determined the claim and project you can proceed to troubleshoot. For example, if the problem was with the monitoring stack itself:

```
oc project openshift-monitoring
```

Isolate the problematic pod:

```
oc logs alertmanager-main-0
```

Review the logs for any strange happenings. Additionally, take a look at the event logs in the project:

```
oc get events --sort-by=.metadata.creationTimestamp
```

You can `rsh` into the container

Start with the `df` commands. You should check for free space as well as inodes free. Usually, there are alerts for both space and inode usage for PVs in the (openshift-.*|kube-.*|default|logging) projects which would indicate the problem. However, sometimes these alerts are missed or do not fire as expected. Use the following commands to verifiy (output has been truncated):

```
df -i
Filesystem                             Inodes  IUsed    IFree IUse% Mounted on
overlay                              41672128 123688 41548440    1% /
tmpfs                                 1020227      5  1020222    1% /etc/alertmanager/config
tmpfs                                 1020227      5  1020222    1% /etc/alertmanager/secrets/alertmanager-main-proxy
tmpfs                                 1020227      5  1020222    1% /etc/alertmanager/secrets/alertmanager-kube-rbac-proxy
tmpfs                                 1020227      7  1020220    1% /etc/alertmanager/secrets/alertmanager-main-tls


df -h
Filesystem                            Size  Used Avail Use% Mounted on
overlay                                80G   16G   64G  20% /
tmpfs                                 3.9G  4.0K  3.9G   1% /etc/alertmanager/config
tmpfs                                 3.9G  4.0K  3.9G   1% /etc/alertmanager/secrets/alertmanager-main-proxy
tmpfs                                 3.9G  4.0K  3.9G   1% /etc/alertmanager/secrets/alertmanager-kube-rbac-proxy
tmpfs                                 3.9G  8.0K  3.9G   1% /etc/alertmanager/secrets/alertmanager-main-tls
```

In some cases, such as the `openshift-monitoring` project, it may be more useful to use AlterManager or Prometheus UIs in order to determine if there are excessive amount of alerts being generated or more metrics gathered than anticipated.

If this is normal growth, a request for additional storage should be submitted in order to rectify the situation.

# Verification

Log into the pods and verify that the additional space can be seen from the applications' perspective. You can use the same `df` commands referenced above.

In rare cases the application pods may need to be killed so that they remount the storage in order to reflect the changes.

```
oc delete pod alertmanager-main-0
```

Once the pod can see the proper amount of storage, wait 15 minutes for the alert to clear.
