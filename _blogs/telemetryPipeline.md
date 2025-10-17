---
published: false
date: '2025-07-01 17:51 +0200'
title: Building a SONiC gNMI Telemetry Pipeline with InfluxDB and Grafana
author: Alexei Kiritchenko
excerpt: Building a SONiC gNMI Telemetry Pipeline with InfluxDB and Grafana
position: hidden
tags:
  - sonic
  - gnmi
  - telemetry
  - InfluxDB
  - Grafana
  - Telegraf
---

{% include toc %}

## Introduction

Network operations demand real-time visibility into device. With the streaming telemetry, it is possible to move beyond traditional polling methods and access detailed metrics straight from the network operating system. 

In this guide, we’ll walk through the process of building a telemetry pipeline for SONiC using gNMI, Telegraf, InfluxDB, and Grafana:

 - **gNMI** will serve as the protocol to subscribe to SONiC operational data.
 - **Telegraf** will act as the collector, parsing and forwarding the data.
 - **InfluxDB** will store the time-series telemetry metrics for analysis.
 - **Grafana** will provide visualization and dashboarding capabilities.
 


## InfluxDB & Telegraf Installation

install InfluxDB

```
sudo docker pull influxdb
```

Install telegraf

```
sudo docker pull telegraf
```

# Launching InfluxDB

InfluxDB 2.x stores its data, configuration, and meta-information in `/root/.influxdb2` by default when running as a standalone process in Docker or a standard environment. This is the internal path within the container where InfluxDB looks for and writes its data. Mapping a host directory to this path ensures that any data written remains intact across container lifecycles.

```
sudo docker run \
    -d --name=influxdb \
    -p 8086:8086 \
    -v  /home/cisco/telemetry/influxdb:/root/.influxdb2 \
    influxdb
```

InfluxDB container listens on port 8086

```
cisco@8ktme:~/telemetry$ sudo docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED          STATUS          PORTS                                       NAMES
458390e551c4   influxdb   "/entrypoint.sh infl…"   20 seconds ago   Up 19 seconds   0.0.0.0:8086->8086/tcp, :::8086->8086/tcp   influxdb
```


Login to Influx DB to set it up:

- `http://<IP>:8086/`
  - or
- `docker exec -it influxdb influx setup`

terminal example:

 - user: cisco
 - password: sonic4lab


```
cisco@8ktme:~/telemetry$ docker exec -it influxdb influx setup
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.47/containers/influxdb/json": dial unix /var/run/docker.sock: connect: permission denied
cisco@8ktme:~/telemetry$ sudo docker exec -it influxdb influx setup
> Welcome to InfluxDB 2.0!
? Please type your primary username cisco
? Please type your password *********
? Please type your password again *********
? Please type your primary organization name cisco
? Please type your primary bucket name sonic
? Please type your retention period in hours, or 0 for infinite 96
? Setup with these parameters?
  Username:          cisco
  Organization:      cisco
  Bucket:            sonic
  Retention Period:  96h0m0s
 Yes
User	Organization	Bucket
cisco	cisco		sonic
```


Create a token to access InfluxDB. Write it down, it will be used later by Telegraf

```
sudo docker exec -it influxdb influx auth create \
  --org cisco \
  --all-access
```

```
root@8ktme:~# sudo docker exec -it influxdb influx auth create \
  --org cisco \
  --all-access
ID			Description	Token								User Name	User ID			Permissions
0e5af3f0ac5ba000			b6bYQOW25oFqTlWFeamT1xiNRrgwreB6AE-pnqULJJla5QlRjPDyPuDHLqDPgf6mH4LGxDaN4C0HPFIAaj-PCQ==	cisco		0e5af3e2d39ba000	[read:orgs/92f4f749a998c872/authorizations write:orgs/92f4f749a998c872/authorizations read:orgs/92f4f749a998c872/buckets write:orgs/92f4f749a998c872/buckets read:orgs/92f4f749a998c872/dashboards write:orgs/92f4f749a998c872/dashboards read:/orgs/92f4f749a998c872 read:orgs/92f4f749a998c872/sources write:orgs/92f4f749a998c872/sources read:orgs/92f4f749a998c872/tasks write:orgs/92f4f749a998c872/tasks read:orgs/92f4f749a998c872/telegrafs write:orgs/92f4f749a998c872/telegrafs read:/users/0e5af3e2d39ba000 write:/users/0e5af3e2d39ba000 read:orgs/92f4f749a998c872/variables write:orgs/92f4f749a998c872/variables read:orgs/92f4f749a998c872/scrapers write:orgs/92f4f749a998c872/scrapers read:orgs/92f4f749a998c872/secrets write:orgs/92f4f749a998c872/secrets read:orgs/92f4f749a998c872/labels write:orgs/92f4f749a998c872/labels read:orgs/92f4f749a998c872/views write:orgs/92f4f749a998c872/views read:orgs/92f4f749a998c872/documents write:orgs/92f4f749a998c872/documents read:orgs/92f4f749a998c872/notificationRules write:orgs/92f4f749a998c872/notificationRules read:orgs/92f4f749a998c872/notificationEndpoints write:orgs/92f4f749a998c872/notificationEndpoints read:orgs/92f4f749a998c872/checks write:orgs/92f4f749a998c872/checks read:orgs/92f4f749a998c872/dbrp write:orgs/92f4f749a998c872/dbrp read:orgs/92f4f749a998c872/notebooks write:orgs/92f4f749a998c872/notebooks read:orgs/92f4f749a998c872/annotations write:orgs/92f4f749a998c872/annotations read:orgs/92f4f749a998c872/remotes write:orgs/92f4f749a998c872/remotes read:orgs/92f4f749a998c872/replications write:orgs/92f4f749a998c872/replications]
```
