# PagerDuty Alert ID: KubeControllerManagerDown

# Description

This alert fires when the Kube Controller Manager pods are down.

## Severity and Impact
Many features stop working when kube-controller-manager is down.

This includes:

* workload controllers (Deployment, ReplicaSet, DaemonSet, ...)
* resource quotas
* pod disruption budgets
* garbage collection
* certificate signing requests
* service accounts, tokens
* storage
* nodes' statuses and taints
* SCCs
* and more...

# Investigation and Triage
The first thing to determine which pod may be having the problem. If the alert doesn't specify the pod, you will have to log into the OpenShift cluster. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-controller-manager` project:

```
oc project openshift-controller-manager
```

You should have a scheduler for each of the control plane:

```
oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
controller-manager-2d6jp   1/1     Running   0          4d3h
controller-manager-lkt4q   1/1     Running   0          4d3h
controller-manager-tglj9   1/1     Running   0          4d3h
```

You can investigate the events:

```
oc get events --sort-by=.metadata.creationTimestamp
```

If there is nothing obvious in the event logs, go pod by pod and investigate the pod logs:

You can also get the logs of each pod:

```
oc logs controller-manager-2d6jp|less
```

If nothing is immediately obvious, start by investigating the underlying storage. There are several storage issues that could trigger this alert.

1. The backend storage for Prometheus or Alertmanager can be full. When this occurs the state of an alert may not be able to be cleared. That means that if communication is temporarily lost, it is possible that the restoration message could not be recorded and the alert will trigger falsely
2. The backend storage for the host running the pods can also impact this alert. If the VM disk is being thrashed, it is possible that the pod will not respond to Prometheus' queries and therefore Prometheus will assume the pod is down.

## Prometheus Storage

You can quickly check from the pod itself, whether its' storage is full:

```
oc rsh prometheus-k8s-0
```

You can either run the `df -h` or `mount` command to determine the status of the persistant mount. 

## Drain and Reboot Problem Node

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
Verify that the alert has cleared from Alertmanager once the root cause has been identified.