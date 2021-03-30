# PagerDuty Alert ID: KubeStateMetricsListErrors
# Description

This fires when the average number of errors over 5 minutes divided by the total number of metrics is greater than 1%. This error is triggered when Prometheus is unable to list the metrics of Kubebernetes states. These metrics are derived from the Kubernetes API and tracks the state of objects in the cluster.

API Issues can be caused but (but not limited to) etcd timeouts, server overloaded, ntp without synchronization, network throughput issues, storage latency issues or other issues.

# Investigation and Triage

Usually this error is accompanied by other API related issues. As such the troubleshooting steps for this alert are similar to other API problems. First verify the issue by logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-kube-apiserver` project:

```
oc project openshift-kube-apiserver
```

There are 3 pods that make up the AlertManager cluster. The pod names are:

```
kube-apiserver-master-0.lab-cluster.ocp4.lab 
kube-apiserver-master-1.lab-cluster.ocp4.lab
kube-apiserver-master-2.lab-cluster.ocp4.lab
```

Make sure to review the logs from each one of the pods:

```
oc logs kube-apiserver-master-0.lab-cluster.ocp4.lab 
```

Also take a look at the events in the project:

```
oc get events --sort-by=.metadata.creationTimestamp
```

You can try restarting the `kubelet` service as well if there is nothing obvious in the events or logs.

Log into a host and restart the service. Since the kubelet service will kill a debug pod, SSH is the recommended way to restart the process

```
ssh core@master-0.lab-cluster.ocp4.lab

sudo systemctl restart kubelet.service
```

You should proceed to troubleshoot the underlying KVM hardware. Ensure that there is enough ram on the host. Identify the correct KVM host and SSH to the box. Once logged in run

```
free -m
```

To get a live picture of the system as a whole you can run `glances`. This tool provides disk activity along with system activity (similar) to `top` all in one interface. You will want to make sure your terminal a decent size to be able to take advantage of `glances`.

If the KVM host is not experiencing performance issues, review alerts and see if there are any `etcd` alerts firing. If they are, these are most likely to be the cause. Performance issues in `etcd` are the most likely cause for API alerts.

You can investigate the `etcd` pods for signs of trouble.

```
oc logs etcd-master-0.lab-cluster.ocp4.lab -c etcd -n openshift-etcd


2020-12-17 21:15:14.065641 W | etcdserver: read-only range request "key:\"/kubernetes.io/namespaces/openshift-kube-controller-manager\" " with result "range_response_count:1 size:1158" took too long (202.040532ms) to execute
2020-12-17 21:15:14.065860 W | etcdserver: read-only range request "key:\"/kubernetes.io/secrets/openshift-apiserver/encryption-config-0\" " with result "range_response_count:0 size:7" took too long (167.400187ms) to execute
```

Please see the ETCD runbooks for further troubleshooting.

# Verification