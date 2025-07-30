---
published: false
date: '2025-07-30 14:26 +0200'
title: Self Signed Certificate for gNMI
author: Alexei Kiritchenko
excerpt: Self Signed Certificate for SONiC gNMI
tags:
  - cisco
  - sonic
  - gnmi
position: hidden
---

{% include toc %}

When setting up secure connections, certificates are essential for encrypting data and verifying identities. Self-signed certificates are not issued by a Certificate Authority. Instead, they are created and signed by the same entity that will use them. Self-signed certificates are easy to create and are free, making them a popular choice for testing, internal networks, or lab environments.

In this article we'll generate certificates directly on a SONiC box.

## Steps to Generate and Deploy Self-Signed Certificates for SONiC gNMI

- Check the Default Certificate Paths in SONiC Configuration
- Create a Working Directory for Certificates
- Generate a Root Self-Signed CA Certificate and Private Key
- Create a Node Private Key and Certificate Signing Request
- Set Read Permissions on the Node Private Key
- CreateSubject Alternative Name (SAN)
- Sign the Node Certificate with the Root CA
- Verify Subject Alternative Names in the Node Certificate
- Copy Certificates and Key to SONiC Telemetry Directory
- Restart the gNMI Docker Container
- Set Up a Directory for Certificates on the gNMI Server
- Copy Certificates from Node to gNMI Server



### SONiC gNMI default configuration


GNMI config is located in `/etc/sonic/config_db.json`, where the default certificates' path can be seen

```
  "GNMI": {
    "certs": {
      "ca_crt": "/etc/sonic/telemetry/dsmsroot.cer",
      "server_crt": "/etc/sonic/telemetry/streamingtelemetryserver.cer",
      "server_key": "/etc/sonic/telemetry/streamingtelemetryserver.key"
    },
    "gnmi": {
      "client_auth": "true",
      "log_level": "2",
      "port": "50051"
    }
  },
```

Create certificates first in the SONiC home folder and then copy them to the gNMI server and to `/etc/sonic/telemetry/`


