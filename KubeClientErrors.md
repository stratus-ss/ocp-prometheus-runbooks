# PagerDuty Alert ID: KubeClientErrors
# Description
This fires when the platform uses a rest client to test the API but receives an http 500+ error. If more than 1% of the tries over 5 minutes fails this alert is triggered. 

# Investigation and Triage

This alert is most often caused by network related latency. However, high disk activity or CPU/RAM contention can also, on occassion, cause the rest API to reply slower than expected. This error is often seen at the same time as KubeAPIErrorBudgetBurn. As these two alerts are similar, their troubleshooting steps share common approaches.

First verify the issue by logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-kube-apiserver` project:

```
oc project openshift-kube-apiserver
```

There are 3 pods that make up the AlertManager cluster. The pod names are:

```
kube-apiserver-master-0.lab-cluster.ocp4.lab 
kube-apiserver-master-1.lab-cluster.ocp4.lab
kube-apiserver-master-2.lab-cluster.ocp4.lab
```

View the logs for each of the pods:

```
oc logs kube-apiserver-master-0.lab-cluster.ocp4.lab -c kube-apiserver
```

You are looking for anything that would indicate an error. The words `Error` or `error` may occur but also `Warning`, `warning` or `connection` are worth searching for. The key to tracking this down is being able to identify the approximate time that the error was generated and look for similar times in the logs.

One of the steps you can try is to restart the `kubelet` service on the nodes.

Log into a host and restart the service. Since the kubelet service will kill a debug pod, SSH is the recommended way to restart the process

```
ssh core@master-0.lab-cluster.ocp4.lab

sudo systemctl restart kubelet.service
```


# Verification
Verification includes making sure the alert is no longer firing. If you identified errors in previous investigations, make sure the errors are no longer present.