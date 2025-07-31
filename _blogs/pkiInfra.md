---
published: true
date: '2025-07-31 14:26 +0200'
title: Building Public Key Infrastructure
author: Alexei Kiritchenko
excerpt: Building Public Key Infrastructure in the lab environment
tags:
  - cisco
  - sonic
  - gnmi
position: hidden
---

{% include toc %}


The Public key infrastructure (PKI) is used to create, distribute, mange digital certificates and keys. In this article we go over steps and deploying your own PKI lab infrastructure

In this POC a linux server will serve as the CA root point. To ensure security, keys will be stored in the `/root/ca/private/` directory with permissions set to 600, restricting access to the root user only.



## Steps on building your own Building Public Key lab Infrastructure

Steps on building your own Building Public Key lab Infrastructure

- Create Self-Signed CA Root certificate
- Add SONiC devices to PKI
- Configure DNS server on SONiC
- Install Root CA
- Generate a private key and Certificate Signing Request on SONiC
- Sign CSR
- Copy the signed certificate back to the router
- Get the key file from a router
- Test gNMI


## Create Self-Signed CA Root certificate



