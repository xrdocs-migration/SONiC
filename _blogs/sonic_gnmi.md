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

Lets track down the gnmi subsystem and order to understand how it starts and what configuration is expected.

Start by identifying the container associated with gNMI. Since SONiC uses Docker containers for its various subsystems, we can use the `docker ps` command and search for telemetry (old name) or gnmi (new name)

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

Next, inspect gnmi docker container and search for Cmd and/or Entrypoint 

```diff

docker inspect gnmi

"Config": {
            "Hostname": "sonic",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": true,
            "AttachStderr": true,
            "Tty": true,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "SYSLOG_TARGET_IP=127.0.0.1",
                "PLATFORM=x86_64-kvm_x86_64-r0",
                "RUNTIME_OWNER=local",
                "NAMESPACE_ID=",
                "NAMESPACE_PREFIX=asic",
                "NAMESPACE_COUNT=1",
                "DEV=",
                "CONTAINER_NAME=gnmi",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "IMAGENAME=docker-sonic-gnmi",
                "DISTRO=bookworm",
                "DEBIAN_FRONTEND=noninteractive",
                "IMAGE_VERSION=HEAD.0-dirty-20250303.105824"
            ],
+           "Cmd": null,
            "Image": "docker-sonic-gnmi:latest",
            "Volumes": null,
            "WorkingDir": "",
+           "Entrypoint": [
+               "/usr/local/bin/supervisord"
            ],

   ... 


```

Cmd has nothing but Entrypoint launches supervisord, a process control system.

connect to the gNMI container and inspect the processes managed by `supervisord`:

```
docker exec -it gnmi bash
```
  
and, as the name suggest, we'd be most likely interested in gnmi-native:

```diff
supervisorctl status
  
  dependent-startup                EXITED    Jun 27 12:17 AM
  dialout                          RUNNING   pid 20, uptime 7:21:20
+ gnmi-native                      RUNNING   pid 18, uptime 7:21:21
  rsyslogd                         RUNNING   pid 11, uptime 7:21:25
  start                            EXITED    Jun 27 12:17 AM
  supervisor-proc-exit-listener    RUNNING   pid 8, uptime 7:21:27
```

The configuration for `gnmi-native` is specified in `/etc/supervisor/conf.d/supervisord.conf`. Open the file and locate the relevant section:


```diff
less /etc/supervisor/conf.d/supervisord.conf

...


  [program:gnmi-native]
+ command=/usr/bin/gnmi-native.sh
  priority=3
  autostart=false
  autorestart=false
  stdout_logfile=syslog
  stderr_logfile=syslog
  dependent_startup=true
  dependent_startup_wait_for=start:exited

...

```

Next, examine the gnmi-native.sh script to understand how the service is configured and launched:

```
less /usr/bin/gnmi-native.sh
```

and yes, we have the source code on gitHub, we can check the files from there too:  
https://github.com/sonic-net/sonic-buildimage/blob/master/dockers/docker-sonic-gnmi/gnmi-native.sh


This script retrieves variables from the `TELEMETRY_VARS_FILE`, which is a template located at `/usr/share/sonic/templates/telemetry_vars.j2`. 

Here’s the template:

```
cat /usr/share/sonic/templates/telemetry_vars.j2
{
    "certs": {\% if "certs" in GNMI.keys() \%}{{ GNMI["certs"] }}{\% else \%}""{\% endif \%},
    "gnmi" : {\% if "gnmi" in GNMI.keys() \%}{{ GNMI["gnmi"] }}{\% else \%}""{\% endif \%},
    "x509" : {\% if "x509" in DEVICE_METADATA.keys() % }{{ DEVICE_METADATA["x509"] }}{\% else \%}""{\% endif \%}
}
```