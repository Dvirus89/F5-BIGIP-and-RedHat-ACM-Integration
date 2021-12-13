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
Enabling Big IP’s WAF capabilities requires breaking the TLS handshake.
RHACM uses client certificate authentication between the managed cluster and the hub. The client certificate contains the managed cluster’s identity at the CN and Organization fields of the x509 certificate. Breaking the TLS handshake causes the client certificate’s identity parameters to disappear. Thereby making all traffic that comes directly from Big IP unauthenticated.


## The Solution

To solve this problem, alongside it’s WAF capabilities, the Big IP will also act as an authentication proxy. Big IP will receive traffic from the managed cluster who’s going to use the client certificate. Big IP will accept traffic from the managed cluster, break the TLS, and forge a new packet.

The Big IP takes the CN and Organization fields from the managed cluster’s client certificate and places them into the packet’s headers as Impersonate-User and Impersonate-Group accordingly.

<this is a draft / placeholder for Michael :) >
