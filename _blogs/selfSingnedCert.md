---
published: true
date: '2025-08-20 14:26 +0200'
title: Self Signed Certificate for gNMI on SONiC
author: Alexei Kiritchenko
excerpt: Self Signed Certificate for SONiC gNMI
tags:
  - cisco
  - sonic
  - gnmi
position: top
---

{% include toc %}

## Introduction

When setting up secure connections, certificates are essential for encrypting data and verifying identities. Self-signed certificates are not issued by a Certificate Authority. Instead, they are created and signed by the same entity that will use them. Self-signed certificates are easy to create and are free, making them a popular choice for testing, internal networks, or lab environments.

In this article we'll generate certificates directly on a SONiC box c1.sonic.cisco.com with IP 1.18.1.1

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



### SONiC gNMI configuration


GNMI config is located in `/etc/sonic/config_db.json`, where the default certificates' path can be seen. In our case the certificates are located in `/etc/sonic/telemetry/`


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

I am not aware of any `show` commands to verify the used certificates. Let's just look at the running parameters of the telemetry process and check server_crt, server_key and ca_crt values

```
docker exec gnmi ps -ef | grep telemetry
root          24       1  0 Aug06 pts/0    00:34:41 /usr/sbin/telemetry -logtostderr --server_crt /etc/sonic/telemetry/streamingtelemetryserver.cer --server_key /etc/sonic/telemetry/streamingtelemetryserver.key --ca_crt /etc/sonic/telemetry/dsmsroot.cer --port 50051 -v=2 --threshold 100 --idle_conn_duration 5
```

Let's do somehting fancy and print just informaiton we need.

**Note:** the command below was AI generated.
{: .notice--primary}

```
docker exec gnmi ps -ef | \
grep "/usr/sbin/telemetry" | awk '{
    for (i=1; i<=NF; i++) {
        if ($i == "--server_crt") {
            print "--server_crt " $(i+1);
        } else if ($i == "--server_key") {
            print "--server_key " $(i+1);
        } else if ($i == "--ca_crt") {
            print "--ca_crt " $(i+1);
        }
    }
}'
```

The output:
```
--server_crt /etc/sonic/telemetry/streamingtelemetryserver.cer
--server_key /etc/sonic/telemetry/streamingtelemetryserver.key
--ca_crt /etc/sonic/telemetry/dsmsroot.cer
```

### Working Directory

Login to your SONiC device and create a `cert` folder

```
cd
mkdir cert
cd cert
```

### Generate a Root Self-Signed CA Certificate and Private Key


Create a root self-signed CA certificate along with its corresponding private key.

```
sudo openssl req -x509 -newkey rsa:4096 -keyout ./dsmsroot.key \
-out ./dsmsroot.cer -sha256 -days 3650 -nodes -subj '/CN=sonic-lab'
```

### Create a Node Private Key and Certificate Signing Request

Create a node private key and a certificate signing request (CSR)

```
sudo openssl req -new -newkey rsa:4096 -nodes \
-keyout streamingtelemetryserver.key -out streamingtelemetryserver.csr \
-subj "/CN=c1" \
-addext "subjectAltName=DNS:c1.sonic.cisco.com,DNS:c1,IP:1.18.1.1"
```


### Set Read Permissions on the Node Private Key

```diff
  ls -lh
  total 16K
  -rw-r--r-- 1 root root 1.8K Jul 31 11:34 dsmsroot.cer
  -rw------- 1 root root 3.2K Jul 31 11:34 dsmsroot.key
  -rw-r--r-- 1 root root 1.7K Jul 31 11:37 streamingtelemetryserver.csr
+ -rw------- 1 root root 3.2K Jul 31 11:37 streamingtelemetryserver.key
```

add read permissions to the key cert file:

```
sudo chmod a+r streamingtelemetryserver.key
```

```diff
  ls -lh
  total 16K
  -rw-r--r-- 1 root root 1.8K Jul 31 11:34 dsmsroot.cer
  -rw------- 1 root root 3.2K Jul 31 11:34 dsmsroot.key
  -rw-r--r-- 1 root root 1.7K Jul 31 11:37 streamingtelemetryserver.csr
+ -rw-r--r-- 1 root root 3.2K Jul 31 11:37 streamingtelemetryserver.key
```

### CreateSubject Alternative Name

Create a san.ext file and add Subject Alternate Name into it:

```
vim san.ext
```

```
subjectAltName = DNS:c1.sonic.cisco.com, DNS:c1, IP:1.18.1.1
```

### Sign the Node Certificate with the Root CA

Sign a node certificate signed by the CA certificate:

```
sudo openssl x509 -req -in streamingtelemetryserver.csr \
  -CA dsmsroot.cer -CAkey dsmsroot.key -CAcreateserial \
  -out streamingtelemetryserver.cer -days 3650 -sha512 \
  -extfile ./san.ext
```


### Verify Subject Alternative Names in the Node Certificate


Check SAN are in the file `openssl x509 -in streamingtelemetryserver.cer -text -noout | grep -A 1 "Subject Alternative Name"`

```
root@c1:/home/cisco/cert# openssl x509 -in streamingtelemetryserver.cer -text -noout | grep -A 1 "Subject Alternative Name"
            X509v3 Subject Alternative Name:
                DNS:c1.sonic.cisco.com, DNS:c1, IP Address:1.18.1.1

```


### Copy Certificates and Key to SONiC Telemetry Directory

Make sure `/etc/sonic/telemetry/` exists and create it if needed.

Copy the certificates into the expected `/etc/sonic/telemetry/` place

```
sudo cp dsmsroot.cer /etc/sonic/telemetry/
sudo cp streamingtelemetryserver.cer /etc/sonic/telemetry/
sudo cp streamingtelemetryserver.key /etc/sonic/telemetry/
```

### Restart the gNMI Docker Container

Restart gnmi docker container for the router to pick up the new certificates:

```
sudo docker restart gnmi
```


### Set Up a Directory for Certificates on the gNMI Server

Create a certs folder on the gNMI server to keep the certificates from the nodes. 

```
cd
mkdir certs
```

### Copy Certificates from Node to gNMI Server

copy the certificates from the node:


```
scp cisco@c1.sonic.cisco.com:~/cert/dsmsroot.cer RootCA.crt
scp cisco@c1.sonic.cisco.com:~/cert/streamingtelemetryserver.cer c1.sonic.cisco.com.crt
scp cisco@c1.sonic.cisco.com:~/cert/streamingtelemetryserver.key c1.sonic.cisco.com.key
```

## Test gNMI

Having certificates in place on the server, test gNMI getting its capabilities:

```
root@8ktme:/home/cisco/ansible# gnmic -a 1.18.1.1:50051 \
    --tls-ca "./certs/RootCA.crt" \
    --tls-cert "./certs/c1.sonic.cisco.com.crt" \
    --tls-key "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    capabilities
gNMI version: 0.7.0
supported models:
  - openconfig-acl, OpenConfig working group, 1.0.2
  - openconfig-acl, OpenConfig working group,
  - openconfig-sampling-sflow, OpenConfig working group,
  - openconfig-interfaces, OpenConfig working group,
  - openconfig-lldp, OpenConfig working group, 1.0.2
  - openconfig-platform, OpenConfig working group, 1.0.2
  - openconfig-system, OpenConfig working group, 1.0.2
  - ietf-yang-library, IETF NETCONF (Network Configuration) Working Group, 2016-06-21
  - sonic-db, SONiC, 0.1.0
supported encodings:
  - JSON
  - JSON_IETF
  - PROTO
  ```
  
  
## gNMI Articles
  
  1. [SONiC gNMI](../sonic_gnmi)
  2. [Self Signed Certificate for gNMI](../selfSingnedCert)
  3. [Building your own Public Key Infrastructure](../pkiInfra)
  4. [SONiC Certificates Deployment Automation](../sonicCertAutoDeployment)
  5. [Integrating Prometheus with SONiC](../sonic_prometheus)
