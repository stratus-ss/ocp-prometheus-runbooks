# PagerDuty Alert ID: ImageRegistryStorageReconfigured
# Description
This alert fires when the registry storage configuration changes. This has the potential to cause data loss

# Investigation and Triage
The first thing to investigate is what errors may exist. After logging into the cluster with `oc login` you will need to change to the `openshift-image-registry` project:

```
oc project openshift-image-registry
```

Start by reviewing the events:

```
oc get events --sort-by=.metadata.creationTimestamp
```

Next review the log files:

```
oc logs cluster-image-registry-operator-8455f689dd-vslql -c cluster-image-registry-operator |less
```

If the change looks to be legitimate you can view the OpenShift Audit logs to determine who or what made the change. First get the list of the current audit logs:

```
oc adm node-logs --role=master --path=openshift-apiserver/
master-0.lab-cluster.ocp4.lab audit-2020-12-03T15-46-42.342.log
master-0.lab-cluster.ocp4.lab audit-2020-12-06T12-49-34.426.log
master-0.lab-cluster.ocp4.lab audit-2020-12-10T04-26-04.303.log
master-0.lab-cluster.ocp4.lab audit-2020-12-13T00-02-29.943.log
master-0.lab-cluster.ocp4.lab audit-2020-12-15T19-29-23.525.log
master-0.lab-cluster.ocp4.lab audit.log
master-1.lab-cluster.ocp4.lab audit-2020-12-03T15-48-48.684.log
master-1.lab-cluster.ocp4.lab audit-2020-12-06T12-53-08.942.log
master-1.lab-cluster.ocp4.lab audit-2020-12-10T04-08-11.055.log
master-1.lab-cluster.ocp4.lab audit-2020-12-12T23-26-31.124.log
master-1.lab-cluster.ocp4.lab audit-2020-12-15T19-00-12.340.log
master-1.lab-cluster.ocp4.lab audit.log
master-2.lab-cluster.ocp4.lab audit-2020-12-04T17-15-56.966.log
master-2.lab-cluster.ocp4.lab audit-2020-12-08T13-07-05.885.log
master-2.lab-cluster.ocp4.lab audit-2020-12-11T09-37-01.813.log
master-2.lab-cluster.ocp4.lab audit-2020-12-14T05-04-22.443.log
master-2.lab-cluster.ocp4.lab audit.log
```

You might have to search multiple logs to find the correct entry. In this case the following command narrowed down the culprit (the output was run through python for easier readibility ):

```
oc adm node-logs master-2.lab-cluster.ocp4.lab --path=kube-apiserver/audit.log |grep 8455f689dd-vslql |grep delete |head -n 1 |python3 -m json.tool
{
    "kind": "Event",
    "apiVersion": "audit.k8s.io/v1",
    "level": "Metadata",
    "auditID": "92f38f93-ae18-4f07-a69e-b66ce074e824",
    "stage": "ResponseComplete",
    "requestURI": "/api/v1/namespaces/openshift-image-registry/pods/cluster-image-registry-operator-8455f689dd-vslql",
    "verb": "delete",
    "user": {
        "username": "kube:admin",
        "groups": [
            "system:cluster-admins",
            "system:authenticated"
        ],
        "extra": {
            "scopes.authorization.openshift.io": [
                "user:full"
            ]
        }
    },
--- SNIP ---
}
```

From the above output you can tell that the user `kube:admin` or `kubeadmin` deleted the image registry operator which triggered a configuration change. You could also search for `configs.imageregistry.operator.openshift.io` which is the configuration which is typically used to set the image storage.

# Verification
As this alert triggers when there has been a configuration change, there is no additional verification. Verification is done through the investigation steps to determine what has occured. 