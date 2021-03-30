# PagerDuty Alert ID: PrometheusBadConfig

# Description
This alert is triggered any time in a 5 minute period, prometheus is unable to successfully reload its configuration file.

# Investigation and Triage

The first thing to determine is what errors you are receiving in the Prometheus logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the Prometheus pods. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

Get the logs for the container indicated by the alert:

```
oc logs prometheus-k8s-0 -c prometheus
```

You may see a line similar to this:

```
level=error ts=2021-01-06T19:52:56.112Z caller=main.go:578 msg="Error reloading config" err="couldn't load configuration (--config.file=\"/etc/prometheus/config_out/prometheus.env.yaml\"): parsing YAML file /etc/prometheus/config_out/prometheus.env.yaml: yaml: unmarshal errors:\n  line 7: field frank_file not found in type config.plain"
```

From this you can see that `line 7` is the problematic line.

Next you will need to decode the Prometheus configuration file:

```
oc get secret prometheus-k8s -o yaml |grep prometheus.yaml.gz: |grep -v "f:" |awk '{print $2}' |base64 -d - |gunzip > prometheus.yaml
```

If the problem identified in the logs is indeed present in the secret you will have to attempt to ascertain when the change was made. Normally, unless the Prometheus Operator itself makes the change, any configuration issues should be automatically reverted to the known-good version that was shipped with the operator.

In this example you can clearly see that the error is caused by a field called `frank_file`. This was a user error and most likely not caused by the operator. First, you should make sure that the operator pod exists in the project:

```
oc get pods |grep prometheus-operator
```

If there is no pod returned by this search, try setting the replicas to 1 in the deployment:

```
oc patch deployment.apps/prometheus-operator --patch '{"spec":{"replicas": 1}}' --type=merge
```

If the pod does exist, try deleting it and letting it respawn on its' own:

```
oc delete pod prometheus-operator-674ff57b7b-jhd9z
```

After about 2 minutes the operator pod should be marked as running and it should correct the correct the pods above 5-8 minutes.

You also have the option of deleting the problematic pod)(s) after about 2 minutes of the operator being up and running.


# Verification

Verify that all the Prometheus pods are now functioning as expected:

```
oc get pods  |grep prometheus |grep -v adapter
```

You should also see the following line in the logs indicating a successful bootstrap of the monitoring stack:

```
level=info ts=2021-01-06T20:20:22.471Z caller=main.go:762 msg="Completed loading of configuration file" filename=/etc/prometheus/config_out/prometheus.env.yaml
```

The alert will clear in 15 minutes or so.