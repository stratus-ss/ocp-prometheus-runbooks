# PagerDuty Alert ID
# ClusterVersionOperatorDown

This job is considered up when the pod `cluster-version-operator-$UID` in the namespace `openshift-cluster-version` is functioning.

# Investigation and Triage

The first thing to determine is whether the pod. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-cluster-version` project:

```
oc project openshift-cluster-version
```

Verify the IP of the pod:
```
oc get pods -o wide
NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE                            
cluster-version-operator-c9b67c44b-9b6hz   1/1     Running   20         6d21h   10.120.120.50   master-0.lab-cluster.ocp4.lab   
```

If the pod is up check the logs from the pod to see if there are any errors:

```
oc logs cluster-version-operator-c9b67c44b-9b6hz |tail -n 20
```

You can also view the events in the project for other clues. You can sort them by timestamp like so:

```
oc get events --sort-by=.metadata.creationTimestamp
```

As a last resort, you can kill the pod:

```
oc delete pods cluster-version-operator-c9b67c44b-9b6hz
```


# Verification

After the pods have been restarted, make sure the pods are up:

```
oc get pods
```

Then view the events. You should see a LeaderElection notification:

```
oc get events --sort-by=.metadata.creationTimestamp 
LAST SEEN   TYPE     REASON             OBJECT                                          MESSAGE
--- SNIP ---
49s         Normal   Started            pod/cluster-version-operator-c9b67c44b-ghfgd    Started container cluster-version-operator
44s         Normal   LeaderElection     configmap/version                               master-2.lab-cluster.ocp4.lab_4b40696d-2471-48ec-9816-941e77bdd02e became leader
```

At this point if there are no more errors and the pod is healthy, the alert should clear itself up in 10-15 minutes.
