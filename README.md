# F5 BIGIP and Red Hat Advanced Cluster Management (RHACM) - integration 
This document provides a high level overview of the integration solution for F5 BIGIP and Red Hat Advanced Cluster Management (RHACM).
The BIGIP is placed between the RHACM hub cluster and the managed OpenShift cluster to allow vareity of features like:
- Full proxy
- WAF Protection
- L7 Health monitor
- etc

### Why do we need this integration from the first place? great question, keep reading.

## The Issue

Out of the box, integrating the two products is not possible because of the next reasons -
Enabling BIGIP’s WAF capabilities requires breaking the TLS handshake.
RHACM uses client certificate authentication between the managed cluster and the hub. The client certificate contains the managed cluster’s identity at the CN and Organization fields of the x509 certificate. Breaking the TLS handshake causes the client certificate’s identity parameters to disappear. Thereby making all traffic that comes directly from BIGIP unauthenticated.

## The Solution

To solve this problem, alongside it’s WAF capabilities, the BIGIP will also act as an authentication proxy. BIGIP will receive traffic from the managed cluster who’s going to use the client certificate. BIGIP will accept traffic from the managed cluster, break the TLS, and forge a new packet.

The BIGIP takes the CN and Organization fields from the managed cluster’s client certificate and places them into the packet’s headers as Impersonate-User and Impersonate-Group accordingly.

![Alt text](images/solution_diagram.png?raw=true "Solution Diagram")

## Example

This example shows how RHACM's _klusterlet_ client certificate can be transformed into headers that will be used for managed cluster authentication against the hub. The authentication parameters from the client certificate are forwarded as impersonation headers to the hub cluster. The headers are forwarded to the hub together with the _Authorization_ header. The _Authorization_ header contains the JWT access token of a service account on the hub. The service account on the hub is able to perform actions as other users (impersonate). The impersonator service account will be used to perform actions as the managed clusters.

### Creating the impersonator user

In order to create and configure the impersonator user, apply the resources at [impersonation-resources](impersonation-resources) -

```
<managed cluster> $ kustomize build impersonation-resources/ | oc apply -f -
```

After configuring the resources, extract the service account's token.

```
<managed cluster> $ oc sa get-token proxy-authenticator -n authentication-proxy

eyJhbGciOiJSUzI1NiIsImtpZCI6IlpxUHJLU0xtVS1KZS1TaGtMSndadl9wenFKRDRLdThFRm5uODBSc1dtOFkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJhdXRoZW50aWNhdGlvbi1wcm94eSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJwcm94eS1hdXRoZW50aWNhdG9yLXRva2VuLTY3dnAyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNs3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InByb3h5LWF1dGhlbnRpY2F0b3IiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhYdadW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmNDcyZjgzZi03NjA4LTQzYTUtOTBmZi05NDY0MDBiMTQyZDEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6YXV0aGVudGljYXRpb24tcHJveHk6cHJveHktYXV0aGVudGljYXRvciJ9.lyu_y9jNf4X0n7PUlefRUSnEJr4s8Wg--1tiApIOY2F3u2g54g37eEzafRVUHBcaIvzGBbIxD9GR2BuaDguGyo4lv_zKyHADaBaveC1qQH4aUQEf9_4OPkNo0Vjph_WAEmEy4I9z17MTuWxW4Q81Adb9sgvA1SrnkD3ia8PtO9GgHV8cx3JG1eG80S3J9Oe0ioC6_lsNvtuYnt_0efBj-gIMUugLGzUTmHFoO2z9iHinLDXsmqiHwk8RYOgytq2OaKvgglHlOB6Xt4mmCODgoOftmSHDo__Qp5iAl42lthAUeTpBaCisVzIbK_yuVdXhRWwCqcuFyFrIv-eBazxZ76gF-v9BseidEWrZPssEsdKLpRkFP--gV1acXtnh-lErEZau_iFNh2_3LLj-Whl4kXIO_2EUL9qcpeDPu_kSWBP5uxRm3Go59UD52w2xQO3zVRSUrlqDbWkkz_Waqzr0LsgflVbFdnqHgnLSAxpCOXRaUToN5j-sVEqtYwobwc_E6h1qVDZ5X-GybFJemqEXIjmwUESEkEh0R311qLdlpYCt_Jo9PvvDas9CUIQ9L6flRvTX5hNL9JOsZQZz0p4m_EdZZbsc3gWObXmkmZQz5KHqs7jcjOgu8e7AFX2UUpR44jWfj1Mfbu5NcI_UyWznJds0dHBVtsa5_OLiiJ3sSGM
```

The extracted token is used to authenticate to the hub using the impersonator service account.

### Extracting the authentication parameters from Klusterlet

To extract the client certificate used by RHACM's _klusterlet_ agent on the managed cluster run the next command -

```
<managed cluster> $ oc extract secret/hub-kubeconfig-secret --to=- -n open-cluster-management-agent
```

A certificate appears under the `tls.crt` field, an example certificate can be found in [certificate.crt](certificate.crt). Use the certificate to extract the `CN`, and `O` fields. These fields define the _klusterlet_'s identity. To extract the fields run the next command -

```
<managed cluster> $ openssl x509 -in certificate.crt -text -noout

...
        Subject: O = system:open-cluster-management:local-cluster + O = system:open-cluster-management:managed-clusters, CN = system:open-cluster-management:local-cluster:drpdd

...
```

### Simulating a request towards the hub

 In order to check the authentication process , use the impersonator's JWT token and the extracted fields from the client certificate. Send the next request towards to hub cluster-

```
$ curl -k -H "Authorization: Bearer <Impersonator Service Account Access Token>" -H "Impersonate-User: <CN_FIELD>" -H "Impersonate-Group: <ORGANIZATION_FIELD_1>" -H "Impersonate-Group: <ORGANIZATION_FIELD_2>" <HUB_APISERVER_URL>
```

An example of a working command -

```
$ curl -k -H "Authorization: Bearer <Impersonator Service Account Access Token>" -H "Impersonate-User: system:open-cluster-management:local-cluster:k6rqw" -H "Impersonate-Group: system:open-cluster-management:managed-clusters" -H "Impersonate-Group: system:open-cluster-management:local-cluster" https://api.cluster-4108.4108.sandbox370.opentlc.com:6443/apis/cluster.open-cluster-management.io/v1/managedclusters/local-cluster

.... "status":{"allocatable":{"attachable-volumes-aws-ebs":"125","cpu":"53500m","ephemeral-storage":"577352668230","hugepages-1Gi":"0","hugepages-2Mi":"0","memory":"220662120Ki","pods":"1250"},"capacity":{"attachable-volumes-aws-ebs":"125","core_worker":"32","cpu":"56","ephemeral-storage":"626467740Ki","hugepages-1Gi":"0","hugepages-2Mi":"0","memory":"226417000Ki","pods":"1250","socket_worker":"2"},"clusterClaims":[{"name":"id.k8s.io","value":"local-cluster"},{"name":"kubeversion.open-cluster-management.io","value":"v1.21.4+6438632"},{"name":"platform.open-cluster-management.io","value":"AWS"} ...
```

### BIGIP

The above process describes a manual procedure of extracting the fields from a client certificate and forging authentication headers. BIGIP helps the process become automatic by applying the next [irule](Client_Cert_Information_Into_Headers.irule). After applying the _irule_, traffic directed to BIGIP is affected by the policy.
