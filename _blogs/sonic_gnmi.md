---
published: true
date: '2025-06-27 09:01 +0200'
title: SONiC gNMI
author: Alexei Kiritchenko
excerpt: Enabling GNMI on SONiC
position: hidden
tags:
  - sonic
  - gnmi
---

{% include toc %}

## Introduction

gNMI is a widely used protocol that allows users to configure, manage, and stream telemetry data from network devices. It uses gRPC as a reliable transport framework and supports RPCs such as GET, SET, and SUBSCRIBE to allow users to monitor and manage network devices. 

SONiC supports gNMI and allows users to configure and monitor network state using gNMI RPCs. 


### Configuring Telemetry and gNMI on SONiC


Let’s face it, configuring SONiC can sometimes feel like a treasure hunt. You might find yourself searching the internet for configuration examples, diving into forum discussions, or pinging colleagues and experts for advice. While all of this effort is valuable, there’s a better way to cut through the noise.


The good news? SONiC is an open-source platform, which means we have access to its source code and we have right tools and information to get all we need. 


Here we’ll focus on gNMI as our example but the approach we’re about to discuss is not limited to gNMI, it can be applied to any other SONiC SONiC.



#### Connecting the dots

Lets track down the gnmi subsystem and order to understand how it starts what configuration is expected from us

Being on a SONiC system looks at the running containers and search for telemetry (old name) or gnmi (new name)

```diff
  docker ps
  
  CONTAINER ID   IMAGE                                COMMAND                  CREATED        STATUS       PORTS     NAMES
  85ed21a4c5ee   docker-fpm-frr:latest                "/usr/bin/docker_ini…"   44 hours ago   Up 7 hours             bgp
  44e007c60466   docker-snmp:latest                   "/usr/bin/docker-snm…"   3 days ago     Up 7 hours             snmp
  9e30cdf2cb42   docker-platform-monitor:latest       "/usr/bin/docker_ini…"   3 days ago     Up 7 hours             pmon
  c1a38c8d5261   docker-sonic-mgmt-framework:latest   "/usr/local/bin/supe…"   3 days ago     Up 7 hours             mgmt-framework
  9c1d59c28ec7   docker-lldp:latest                   "/usr/bin/docker-lld…"   3 days ago     Up 7 hours             lldp
+ 663da58aec20   docker-sonic-gnmi:latest             "/usr/local/bin/supe…"   3 days ago     Up 7 hours             gnmi
  0cf445c57972   0733e01e7e53                         "/usr/bin/docker_ini…"   3 days ago     Up 7 hours             dhcp_relay
  b5cd2f24b1fd   docker-router-advertiser:latest      "/usr/bin/docker-ini…"   3 days ago     Up 7 hours             radv
  f92130c6a45e   docker-syncd-ciscovs:latest          "/usr/bin/docker_ini…"   3 days ago     Up 7 hours             syncd
  139d2025f43a   docker-teamd:latest                  "/usr/local/bin/supe…"   3 days ago     Up 7 hours             teamd
  2e2d1dee59a2   docker-orchagent:latest              "/usr/bin/docker-ini…"   3 days ago     Up 7 hours             swss
  c7d1b5197f69   docker-eventd:latest                 "/usr/local/bin/supe…"   3 days ago     Up 7 hours             eventd
  14a8ebc09b99   docker-database:latest               "/usr/local/bin/dock…"   3 days ago     Up 7 hours             database
  root@sl1:/home/cisco#
```