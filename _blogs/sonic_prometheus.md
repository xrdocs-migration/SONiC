---
published: true
date: '2025-09-15 17:51 +0200'
title: Integrating Prometheus with SONiC
author: Alexei Kiritchenko
excerpt: monitor SONiC network devices with Prometheus
position: hidden
tags:
  - sonic
  - gnmi
  - telemetry
  - prometheus
---
{% include toc %}

## Introduction

Prometheus is an open-source monitoring system that operates primarily on a pull-based model, scraping metrics from endpoints. However, Prometheus does not natively support the gNMI (gRPC Network Management Interface) streaming protocol, which is increasingly common for managing and monitoring modern network devices like those running SONiC. 

In this article, we demonstrate how to deploy a Prometheus server alongside a Prometheus exporter on a SONiC device to collect telemetry data from the network operating system (NOS). We’ll walk through the process of setting up each component and show how Prometheus can be used to monitor SONiC devices by gathering and visualizing essential network metrics.

## Prometheus installation

In our example, we will install the Prometheus server on a dedicated host with the IP address 172.20.166.81, while the Prometheus exporter will be deployed on the c1 SONiC router. 

Download Prometheus: <https://prometheus.io/download/>

```console
cisco@8ktme:~$ wget https://github.com/prometheus/prometheus/releases/download/v3.0.0-beta.1/prometheus-3.0.0-beta.1.linux-amd64.tar.gz
--2024-10-31 22:33:03--  https://github.com/prometheus/prometheus/releases/download/v3.0.0-beta.1/prometheus-3.0.0-beta.1.linux-amd64.tar.gz
Resolving proxy-wsa.esl.cisco.com (proxy-wsa.esl.cisco.com)... 173.36.240.166
Connecting to proxy-wsa.esl.cisco.com (proxy-wsa.esl.cisco.com)|173.36.240.166|:80... connected.
Proxy request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/6838921/6a74f190-c1b0-4783-b997-385eb78ab59f?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20241031%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241031T223656Z&X-Amz-Expires=300&X-Amz-Signature=3526357bd9d596403e415c5d9d5aef8b316fcd99a0069ec5f90eae710f5f9ed9&X-Amz-SignedHeaders=host&response-content-disposition=attachment%3B%20filename%3Dprometheus-3.0.0-beta.1.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream [following]
--2024-10-31 22:33:04--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/6838921/6a74f190-c1b0-4783-b997-385eb78ab59f?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20241031%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20241031T223656Z&X-Amz-Expires=300&X-Amz-Signature=3526357bd9d596403e415c5d9d5aef8b316fcd99a0069ec5f90eae710f5f9ed9&X-Amz-SignedHeaders=host&response-content-disposition=attachment%3B%20filename%3Dprometheus-3.0.0-beta.1.linux-amd64.tar.gz&response-content-type=application%2Foctet-stream
Connecting to proxy-wsa.esl.cisco.com (proxy-wsa.esl.cisco.com)|173.36.240.166|:80... connected.
Proxy request sent, awaiting response... 200 OK
Length: 112787601 (108M) [application/octet-stream]
Saving to: ‘prometheus-3.0.0-beta.1.linux-amd64.tar.gz’

prometheus-3.0.0-beta.1.linux-amd64 100%[==================================================================>] 107.56M  42.4MB/s    in 2.5s

2024-10-31 22:33:07 (42.4 MB/s) - ‘prometheus-3.0.0-beta.1.linux-amd64.tar.gz’ saved [112787601/112787601]
```

Unpack the downloaded file

```console
cisco@8ktme:~$ tar xfvz prometheus-3.0.0-beta.1.linux-amd64.tar.gz
prometheus-3.0.0-beta.1.linux-amd64/
prometheus-3.0.0-beta.1.linux-amd64/LICENSE
prometheus-3.0.0-beta.1.linux-amd64/NOTICE
prometheus-3.0.0-beta.1.linux-amd64/promtool
prometheus-3.0.0-beta.1.linux-amd64/prometheus.yml
prometheus-3.0.0-beta.1.linux-amd64/prometheus
cisco@8ktme:~$
```

cd into the unpacked folder and launch Prometheus

```console
cd prometheus-3.0.0-beta.1.linux-amd64/
./prometheus&
```

Server is run on port :9090, so in our lab that would be `http://172.20.166.81:9090/query`

![server](../../images/telemetry/image.png)
