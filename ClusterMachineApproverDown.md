> [!WARN]
> This alert is no longer valid in OCP 4.17+


# PagerDuty Alert ID: ClusterMachineApprover Down
# Description

This job is considered up when the pod `machine-approver-$UID` in the namespace `openshift-cluster-machine-approver` is functioning and the service `machine-approver` exists.

## Severity and Impact


# Investigation and Triage


The first thing to determine is whether the pod and service exist. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-cluster-machine-approver` project:

```
oc project openshift-cluster-machine-approver
```

Verify the IP of the pod:
```
oc get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP              NODE                           
machine-approver-569f6d8b5b-hxqm8   2/2     Running   0          6d18h   10.120.120.50   master-0.lab-cluster.ocp4.lab  
```

Check that the service exists and has the correct IP:

```
 oc describe svc machine-approver
Name:              machine-approver
Namespace:         openshift-cluster-machine-approver
--- SNIP ---
Endpoints:         10.120.120.50:9192
```

If the IP and service do not match kill the pod:

```
oc delete pod machine-approver-569f6d8b5b-hxqm8
```

If the pod and service match, check the logs from the pod to see if there are any errors:

```
oc logs machine-approver-569f6d8b5b-hxqm8
```

You can also view the events in the project for other clues. You can sort them by timestamp like so:

```
oc get events --sort-by=.metadata.creationTimestamp
```

# Verification

If there are no errors observed, the next thing to check is that the port is accessible from the outside. As this port is required for VM to VM communication, the pod bonds to the host IP. It is usually one of the Control Plane that host the pod. From a computer on the same subnet, you can test connectivity like this:

```
> /dev/tcp/$HOST_IP/9192
```
If your prompt returns, the port is open. Otherwise you will receive the following error:

```
-bash: connect: Connection refused
-bash: /dev/tcp/10.120.120.50/9192: Connection refused
```

Now that the container and service are accepting connections the alert should clear within 10-15m.