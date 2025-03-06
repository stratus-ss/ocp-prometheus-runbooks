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
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.17.11   True        False         False      3h27m   
baremetal                                  4.17.11   True        False         False      33d     
cloud-controller-manager                   4.17.11   True        False         False      33d     
cloud-credential                           4.17.11   True        False         False      34d     
cluster-autoscaler                         4.17.11   True        False         False      33d     
config-operator                            4.17.11   True        False         False      33d     
console                                    4.17.11   True        False         False      33d     
control-plane-machine-set                  4.17.11   True        False         False      33d     
csi-snapshot-controller                    4.17.11   True        False         False      33d     
dns                                        4.17.11   True        False         False      24d     
etcd                                       4.17.11   True        False         False      33d     
image-registry                             4.17.11   True        False         False      33d     
ingress                                    4.17.11   True        False         False      3h47m   
insights                                   4.17.11   True        False         False      33d     
kube-apiserver                             4.17.11   True        False         False      33d     
kube-controller-manager                    4.17.11   True        False         False      33d     
kube-scheduler                             4.17.11   True        False         False      33d     
kube-storage-version-migrator              4.17.11   True        False         False      23d     
machine-api                                4.17.11   True        False         False      33d     
machine-approver                           4.17.11   True        False         False      33d     
machine-config                             4.17.11   True        False         False      33d     
marketplace                                4.17.11   True        False         False      33d     
monitoring                                 4.17.11   True        False         False      33d     
network                                    4.17.11   True        False         False      33d     
node-tuning                                4.17.11   True        False         False      33d     
openshift-apiserver                        4.17.11   True        False         False      3h47m   
openshift-controller-manager               4.17.11   True        False         False      33d     
openshift-samples                          4.17.11   True        False         False      33d     
operator-lifecycle-manager                 4.17.11   True        False         False      33d     
operator-lifecycle-manager-catalog         4.17.11   True        False         False      33d     
operator-lifecycle-manager-packageserver   4.17.11   True        False         False      3h47m   
service-ca                                 4.17.11   True        False         False      33d     
storage                                    4.17.11   True        False         False      33d    
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

Review the status of all Operators to discover if multiple Operators are
in a `Degraded` state:

    ```console
    $ oc get clusteroperator
    ```

Review information about the current status of the Operator:

    ```console
    $ oc get clusteroperator $CLUSTEROPERATOR -ojson | jq .status.conditions
    ```

Review the associated resources for the Operator:

    ```console
    $ oc get clusteroperator $CLUSTEROPERATOR -ojson | jq .status.relatedObjects
    ```

Review the logs and other artifacts for the Operator. For example, you can
collect the logs of a specific Operator and store them in a local directory
named `out`:

    ```console
    $ oc adm inspect clusteroperator/$CLUSTEROPERATOR --dest-dir=out
    ```



# Verification

Run 

```
oc get clusteroperator
```

To verify that there are no further degraded operators.