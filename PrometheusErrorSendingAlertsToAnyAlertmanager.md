# PagerDuty Alert ID: PrometheusErrorSendingAlertsToAnyAlertmanager
# Description

This alert fires when Prometheus has encountered more than 3% errors sending alerts to AlertManager.

# Investigation and Triage

As the communication between Prometheus and AlertManager are internal to the cluster by default the most common cause of this alert is that one or both components are having problems. 

The first thing to determine is what errors you are receiving in the Prometheus logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the Prometheus pods. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

Check the status in the project as well as the running states of the pods:

```
oc get events --sort-by=.metadata.creationTimestamp
oc get pods
```

If there are no immediate indications of a problem you will need to start looking at the logs of each of the pods. View the logs for each pod:

```
oc logs -c prometheus prometheus-k8s-0 
oc logs -c prometheus prometheus-k8s-1
```

Then view the logs for AlertManager:

```
oc logs alertmanager-main-0 -c alertmanager
oc logs alertmanager-main-1 -c alertmanager
oc logs alertmanager-main-2 -c alertmanager
```

It is also possible there are problems in the `alertmanager-proxy` pods as well. While it is rare, there is a possibility that one or more of the pods enter a 'stuck' state where they are in fact running according to OpenShift but the processes inside the pod are misbehaving. Try restarting the AlertManager pods one at a time with about a 3 minute delay inbetween each restart.

```
oc delete pod alertmanager-main-0
```


# Verification

The verification for this would be to use the following Prometheus Query

```
rate(prometheus_notifications_errors_total{job="prometheus-k8s",namespace="openshift-monitoring"}[5m])
```

To watch the number of errors over time. You can adjust the `[5m]` argument to say `[1m]` in order to get closer to a live view of the errors that may still be generated. In a healthy cluster the result should be a low value in the output. Zero is ideal.

When the number of errors has recovered, wait 15 minutes for the alert to clear.