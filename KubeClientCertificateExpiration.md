# PagerDuty Alert ID: KubeClientCertificateExpiration
# Description
This alert triggers as a warning 90 minutes before certificate expiration and as a critical alert 60 minutes before certificate expiry.
# Investigation and Triage

The first thing to determine is whether the SSL Certificate is actually expired. After logging into the cluster with `oc login` you will need to examine the `kube-apiserver-operator-serving-cert`. Change to the `openshift-kube-apiserver-operator` project:

```
oc project openshift-kube-apiserver-operator
```

Extract the secrets' base64 SSL certificate and look for the Validity

```
oc get secrets kube-apiserver-operator-serving-cert -o yaml |grep 'tls.crt' |awk '{print $2}' |base64 -d - |openssl x509 -text -noout -in - |grep Issuer -A 3

        Issuer: CN = openshift-service-serving-signer@1606852393
        Validity
            Not Before: Dec  1 19:53:32 2020 GMT
            Not After : Dec  1 19:53:33 2022 GMT
```

If the certificate is infact invalid and managed by OpenShift platform, as of OCP 4.4.8 OpenShift the platform will rotate the certificates. You should see pending CSRs:

```
oc get csr
```

Always validate that the CSR is being made from a known, trusted source:

```
oc describe csr <csr_name> 
```

Then approve the valid CSRs:

```
oc adm certificate approve <csr_name>
```

# Verification
This alert should clear itself within 10 minutes once all of the new CSRs are approved. 