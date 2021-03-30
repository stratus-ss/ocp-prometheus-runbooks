# PagerDuty Alert ID
# ClusterOperatorDown
# Investigation and Triage

The first thing to determine is whether the alert is accurate and which operators are currently down. After logging into the cluster with `oc login` you will need to examine each of the running operators to determine which container is having the problem. 

You can run
```
oc get clusteroperator
```

A healthy cluster will return the following results:

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.20    True        False         False      11d
cloud-credential                           4.5.20    True        False         False      12d
cluster-autoscaler                         4.5.20    True        False         False      12d
config-operator                            4.5.20    True        False         False      12d
console                                    4.5.20    True        False         False      11d
csi-snapshot-controller                    4.5.20    True        False         False      34h
dns                                        4.5.20    True        False         False      12d
etcd                                       4.5.20    True        False         False      12d
image-registry                             4.5.20    True        False         False      12d
ingress                                    4.5.20    True        False         False      12d
insights                                   4.5.20    True        False         False      12d
kube-apiserver                             4.5.20    True        False         False      12d
kube-controller-manager                    4.5.20    True        False         False      12d
kube-scheduler                             4.5.20    True        False         False      12d
kube-storage-version-migrator              4.5.20    True        False         False      63m
machine-api                                4.5.20    True        False         False      12d
machine-approver                           4.5.20    True        False         False      12d
machine-config                             4.5.20    True        False         False      9h
marketplace                                4.5.20    True        False         False      6d17h
monitoring                                 4.5.20    True        False         False      62m
network                                    4.5.20    True        False         False      12d
node-tuning                                4.5.20    True        False         False      11d
openshift-apiserver                        4.5.20    True        False         False      62m
openshift-controller-manager               4.5.20    True        False         False      34h
openshift-samples                          4.5.20    True        False         False      11d
operator-lifecycle-manager                 4.5.20    True        False         False      12d
operator-lifecycle-manager-catalog         4.5.20    True        False         False      12d
operator-lifecycle-manager-packageserver   4.5.20    True        False         False      9h
service-ca                                 4.5.20    True        False         False      12d
storage                                    4.5.20    True        False         False      11d
```

If one of these is failing you can run the following on specific operators

```
oc describe $CLUSTEROPERATOR
```

At the end of the output you can find the `Related Objects` section and the `Events` section will will give you clues as to the next line of troubleshooting.

You can also see if there are any events in the namespace:

```
oc get events --sort-by=.metadata.creationTimestamp -n openshift-cluster-version
```



# Verification

Run 

```
oc get clusteroperator
```

To verify that there are no further degraded operators.