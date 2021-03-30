# PagerDuty Alert ID
# AlertManager Members Inconsistent
# Investigation and Triage

This alert will fire when the service endpoints do not match the current IPs of the individual pods.

The first thing to determine is what IPs in the service endpoint should be. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

There are 3 pods that make up the AlertManager cluster. The pod names are:

```
alertmanager-main-0
alertmanager-main-1
alertmanager-main-2
```

Take a look at the Service endpoints (output truncated):

```
oc describe svc alertmanager-main
Name:              alertmanager-main
Namespace:         openshift-monitoring
--- SNIP ---
Endpoints:         10.128.2.3:9095,10.129.2.5:9095,10.131.0.44:9095
```


Next verify the pod IPs:

```
oc get pods -o wide |grep alert

alertmanager-main-0                            5/5     Running   0          11d     10.129.2.6
alertmanager-main-1                            5/5     Running   0          2d16h   10.131.0.44     
alertmanager-main-2                            5/5     Running   0          6d      10.128.2.3      
```

Delete the problem pod:

```
oc delete pod alertmanager-main-0
```

The pods should now reconcile with the service.

# Verification

After deleting the pod, ensure that the IP has changed:

```
oc get pods -o wide |grep alert
alertmanager-main-0                            5/5     Running   0          61s     10.129.2.29     
alertmanager-main-1                            5/5     Running   0          2d16h   10.131.0.44     
alertmanager-main-2                            5/5     Running   0          6d1h    10.128.2.3      
```

Once the pod's IP has changed validate the service endpoint

```
oc describe svc alertmanager-main
Name:              alertmanager-main
Namespace:         openshift-monitoring
--- SNIP ---
Endpoints:         10.128.2.3:9095,10.129.2.29:9095,10.131.0.44:9095
```

The alert should now resolve within 7-10 minutes.