# PagerDuty Alert ID: KubeAPIErrorBudgetBurn
# Description
This alert tracks both slow requests and errors over time. This alert means either an increase of errors, slow requests or both over time.

## Severity & Impact

These alerts are triggered when the Kubernetes API server is encountering:

* Many 5xx failed requests and/or
* Many slow requests

The urgency of these alerts is determined by the values of their long and short labels:

Critical
 * **long**: 1h and *short*: 5m: less than ~2 days -- You should fix the problem as soon as possible!
 * **long**: 6h and *short*: 30m: less than ~5 days -- Track this down now but no immediate fix required.


# Investigation and Triage

Usually the API server itself is not the issue. However, first verify the issue by logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-kube-apiserver` project:

```
oc project openshift-kube-apiserver
```

There are 3 pods that make up the AlertManager cluster. The pod names are:

```
kube-apiserver-master-0.lab-cluster.ocp4.lab 
kube-apiserver-master-1.lab-cluster.ocp4.lab
kube-apiserver-master-2.lab-cluster.ocp4.lab
```

If you do not see the `OpenAPI spec does not exist` error or restarting the kubelet service it's likely that ETCD is performing poorly.

You can investigate the `etcd` pods for signs of trouble.

```
oc logs etcd-master-0.lab-cluster.ocp4.lab -c etcd -n openshift-etcd


2020-12-17 21:15:14.065641 W | etcdserver: read-only range request "key:\"/kubernetes.io/namespaces/openshift-kube-controller-manager\" " with result "range_response_count:1 size:1158" took too long (202.040532ms) to execute
2020-12-17 21:15:14.065860 W | etcdserver: read-only range request "key:\"/kubernetes.io/secrets/openshift-apiserver/encryption-config-0\" " with result "range_response_count:0 size:7" took too long (167.400187ms) to execute
```

## Checking Scoped API queries

The following queries can be used individually to determine the resource kinds
that contribute to the SLO violation once the main contributor has been
identified.

`error`:
```sh
sum by(resource) (rate(apiserver_request_total{job="apiserver",verb=~"LIST|GET",code=~"5.."}[1d]))
/ scalar(sum(rate(apiserver_request_total{job="apiserver",verb=~"LIST|GET"}[1d])) or vector(0))
```

`slow-resource`:
```sh
sum by(resource) (rate(apiserver_request_duration_seconds_count{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec",scope="resource"}[1d]))
-
(sum by(resource) (rate(apiserver_request_duration_seconds_bucket{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec",scope="resource",le="0.1"}[1d])) or vector(0))
/ scalar(sum(rate(apiserver_request_total{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec"}[1d])))
```

`slow-namespace`:
```sh
sum by(resource) (rate(apiserver_request_duration_seconds_count{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec",scope="namespace"}[1d]))
-
(sum by(resource) (rate(apiserver_request_duration_seconds_bucket{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec",scope="namespace",le="0.5"}[1d])) or vector(0))
/ scalar(sum(rate(apiserver_request_total{job="apiserver",verb=~"LIST|GET",subresource!~"proxy|log|exec"}[1d])))
```

`slow-cluster`:
```sh
sum by(resource) (rate(apiserver_request_duration_seconds_count{job="apiserver",verb=~"LIST|GET",scope="cluster"}[1d]))
-
(sum by(resource) (rate(apiserver_request_duration_seconds_bucket{job="apiserver",verb=~"LIST|GET",scope="cluster",le="5"}[1d])) or vector(0))
/ scalar(sum(rate(apiserver_request_total{job="apiserver",verb=~"LIST|GET"}[1d])))
```


Please see the ETCD runbooks for further troubleshooting. Specifically either slow gRPC requests of the High FSync duration. 

## Mitigations

### Restart the kubelet (after upgrade)

If these alerts are triggered following a cluster upgrade, try restarting the
kubelets per description [here][5420801].

### Determine the source of the error or slow requests

There isn't a straightforward way to identify the root causes, but in the past
we were able to narrow down bugs by examining the failed resource counts in the
audit logs.

Gather the audit logs of the cluster:

```sh
oc adm must-gather -- /usr/bin/gather_audit_logs
```

Install the [cluster-debug-tools][cluster-debug-tools] as a kubectl/oc plugin.

Use the `audit` subcommands to gather information on users that sends the
requests, the resource kinds, the request verbs etc.

E.g. to determine who generate the `apirequestcount` resource slow requests and
what these requests are doing:

```sh
oc dev_tool audit -f ${kube_apiserver_audit_log_dir} -otop --by=user resource="apirequestcounts"

oc dev_tool audit -f ${kube_apiserver_audit_log_dir} -otop --by=verb resource="apirequestcounts" --user=${top-user-from-last-command}
```

The `audit` subcommand also supports the `--failed-only` option which can be
used to return failed requests only:

```sh
# find the top-10 users with the highest failed requests count
oc dev_tool audit -f ${kube_apiserver_audit_log_dir} --by user --failed-only -otop

# find the top-10 failed API resource calls of a specific user
oc dev_tool audit -f ${kube_apiserver_audit_log_dir} --by resource --user=${service_account} --failed-only -otop

# find the top-10 failed API verbs of a specific user on a specific resource
oc dev_tool audit -f ${kube_apiserver_audit_log_dir} --by verb --user=${service_account} --resource=${resources} --failed-only -otop
```

# Verification
Once the problems have been resolved wait 15m for the alert to clear.