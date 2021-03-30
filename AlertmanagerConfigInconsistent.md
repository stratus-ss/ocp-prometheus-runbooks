# PagerDuty Alert ID: AlertManager's Configuration Is Not Consistent
# Description
This alert is caused when the AlertManager pods have different configurations. A hash is calculated and if there is more than one unique hash, the configuration is considered inconsistent.


# Investigation and Triage

The first thing to determine is what the expected configuration hash should be. After logging into the cluster with `oc login` you will need to examine each of the running pods to determine which container is having the problem. Change to the `openshift-monitoring` project:

```
oc project openshift-monitoring
```

There are 3 pods that make up the AlertManager cluster. The pod names are:

```
alertmanager-main-0
alertmanager-main-1
alertmanager-main-2
```

You can get the hashes in order by running the following command:

```
for x in 0 1 2; do  oc rsh alertmanager-main-${x}  /usr/bin/sha256sum /etc/alertmanager/alertmanager.yml 2>/dev/null; done
```
This will loop through each pod and get the sha256sum and display them in order:

```
564d8cad2c71dd5d9f27f598dcc0a46ba361da6420180c68003c3fb127126096  /etc/alertmanager/alertmanager.yml
765d8cad2c71dd5d9dacf598dcc0a976d361daabbc431c68003c3fb127121187  /etc/alertmanager/alertmanager.yml
564d8cad2c71dd5d9f27f598dcc0a46ba361da6420180c68003c3fb127126096  /etc/alertmanager/alertmanager.yml
```

This will give you an inidcation of which container is likely to have not reloaded its configuration file. In this case, its `alertmanager-main-1`. You can kill this pod to have it reload the configuration file:

```
oc delete pod --grace-period=0 alertmanager-main-1
```

After deleting the pod you should see the same hashes. Check to make sure the pod has been restarted:

```
oc get pods |grep main

alertmanager-main-0                            5/5     Running   0          8d
alertmanager-main-1                            5/5     Running   0          37s
alertmanager-main-2                            5/5     Running   0          3d8h
```

Rerun the hash check:

```
for x in 0 1 2; do  oc rsh alertmanager-main-${x}  /usr/bin/sha256sum /etc/alertmanager/alertmanager.yml 2>/dev/null; done

564d8cad2c71dd5d9f27f598dcc0a46ba361da6420180c68003c3fb127126096  /etc/alertmanager/alertmanager.yml
564d8cad2c71dd5d9f27f598dcc0a46ba361da6420180c68003c3fb127126096  /etc/alertmanager/alertmanager.yml
564d8cad2c71dd5d9f27f598dcc0a46ba361da6420180c68003c3fb127126096  /etc/alertmanager/alertmanager.yml
```

# Verification
Wait approximately 5 minutes for the alert to clear. Use the AlertManager WebUI to ensure that this alert is no longer firing