# PagerDuty Alert ID: PrometheusRuleFailures
# Description
This alert is designed to monitor both the default `prometheus-k8s` and the `prometheus-user-workload` ruleset. If there any errors produced by either of these rulesets over a 5 minute period, this alert will fire.
# Investigation and Triage
This is a generic alert that will trigger based on *any* of the default rules. As such the troubleshooting techniques are related to which component is throwing the error. Some common causes are:

1. Having multiple storage classes using the same provisioner
2. Multiple instances of Prometheus running in the cluster
3. Custom alerting rule misconfiguration

The first thing to determine is what errors you are receiving in the Prometheus logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the Prometheus pods. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

View the logs for each pod:

```
oc logs -c prometheus prometheus-k8s-0 
oc logs -c prometheus prometheus-k8s-1
```

If there is nothing obvious in the Prometheus pods, you can check the node-exporter pods directly:

```
oc get pods | grep node-exporter

node-exporter-468gg                            2/2     Running   0          18d
node-exporter-c59rt                            2/2     Running   0          18d
node-exporter-ft6xt                            2/2     Running   0          18d
node-exporter-v4jsd                            2/2     Running   0          18d
node-exporter-vfbt5                            2/2     Running   0          18d
node-exporter-z28hf                            2/2     Running   0          18d

oc logs -c node-exporter node-exporter-468gg
```

Repeat the above for each of the `node-exporter-<uid>` pods that exist.

# Verification

As this alert is a symptom of an underlying problem the verification for this alert will be to fix the underlying problem and then wait 15-20 minutes for this alert to clear.
