# PagerDuty Alert ID: PrometheusRemoteStorageFailures
# Description

This alert fires if greater than 1% of the Prometheus sample data is able to be sent to the remote storage. 

# Investigation and Triage

There are two common causes for this error. The first is network latency between OpenShift and the storage host. The second is an actual failure on the storage backend. The storage backend could have high I/O which would account for a failure in the writing of sample data.

The first thing to determine is what errors you are receiving in the Prometheus logs. After logging into the cluster with `oc login` you will need to examine the pod logs for each of the Prometheus pods. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

View the logs for each pod:

```
oc logs -c prometheus prometheus-k8s-0 
oc logs -c prometheus prometheus-k8s-1
```

If there are no other errors present in the logs, verify that the Persistent Volumes are healthy along with their claims:

```
oc get pvc
oc get pv |grep prometheus
```

If, from OpenShift, the storage seems to be functioning normally. You will need to proceed with checking for other storage related alerts for your storage backend.

# Verification

Once you have resolved the storage backend issues, the alert will clear within 15 minutes.

You can run the following Prometheus queries to display the current number of failed samples

```

```