# PagerDuty Alert ID: KubeSchedulerDown
# Description
The Kubernetes scheduler is a control plane process which assigns Pods to Nodes. The scheduler determines which Nodes are valid placements for each Pod in the scheduling queue according to constraints and available resources. The scheduler then ranks each valid Node and binds the Pod to a suitable Node. When this component is down no new work can be scheduled
# Investigation and Triage

The first thing to determine which pod may be having the problem. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-kube-scheduler` project:

```
oc project openshift-kube-scheduler
```

You should have a scheduler for each of the control plane:

```
oc get pods
NAME                                                     READY   STATUS      RESTARTS   AGE
openshift-kube-scheduler-master-0.lab-cluster.ocp4.lab   2/2     Running     4          72m
openshift-kube-scheduler-master-1.lab-cluster.ocp4.lab   2/2     Running     2          72m
openshift-kube-scheduler-master-2.lab-cluster.ocp4.lab   2/2     Running     3         72m
revision-pruner-13-master-0.lab-cluster.ocp4.lab         0/1     Completed   0          12d
revision-pruner-13-master-1.lab-cluster.ocp4.lab         0/1     Completed   0          13d
revision-pruner-13-master-2.lab-cluster.ocp4.lab         0/1     Completed   0          13d
```

You can investigate the events:

```
oc get events --sort-by=.metadata.creationTimestamp
```

You can also get the logs of each pod:

```
oc logs openshift-kube-scheduler-master-1.lab-cluster.ocp4.lab|less
```

Check the VMs for high load, I/O and ram usage on the control plane and the KVM host as well. Using `top` examin the `load average`. If the `load average is greater than the %CPU usage, the host is most likely constrained by disk I/O. Don't forget to check the available ram in both the VM and the KVM host:

```
free -m
```

If the KVM host has overcommitted ram or is ram starved, the VM will not know this and think it has more resources available than it can actually access. This will cause performance issues in the VM.

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

Verify that there have been no further pod restarts:

```
oc get pods
```

Verify that there are no further errors evident in the events:

```
oc get events --sort-by=.metadata.creationTimestamp
```

Wait 15m for the alert to clear and then verify that the alert is no longer firing in the UI.