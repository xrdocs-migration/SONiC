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
  
and, as the name suggest, we'd be most likely interested in `gnmi-native`:

```diff
supervisorctl status
  
  dependent-startup                EXITED    Jun 27 12:17 AM
  dialout                          RUNNING   pid 20, uptime 7:21:20
+ gnmi-native                      RUNNING   pid 18, uptime 7:21:21
  rsyslogd                         RUNNING   pid 11, uptime 7:21:25
  start                            EXITED    Jun 27 12:17 AM
  supervisor-proc-exit-listener    RUNNING   pid 8, uptime 7:21:27
```

The configuration for `gnmi-native` is specified in `/etc/supervisor/conf.d/supervisord.conf`.   Open the file and locate the relevant section:


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



Next, examine the `gnmi-native.sh` script to understand how the service is configured and launched:

```
less /usr/bin/gnmi-native.sh
```


and yes, we have the source code on gitHub, we can check the files from there too:  
<https://github.com/sonic-net/sonic-buildimage/blob/master/dockers/docker-sonic-gnmi/gnmi-native.sh>


This script retrieves variables from the `TELEMETRY_VARS_FILE`, which is a template located at `/usr/share/sonic/templates/telemetry_vars.j2`. 

Here’s the template:

```
cat /usr/share/sonic/templates/telemetry_vars.j2
```

![sonic_gnmi_p1_telemetry_vars.j2.png]({{site.baseurl}}/images/sonic_gnmi_p1_telemetry_vars.j2.png)


TELEMETRY_VARS will be populated running `sonic-cfggen` with this template

```
TELEMETRY_VARS=$(sonic-cfggen -d -t $TELEMETRY_VARS_FILE)
```


From this we can understand the script would be searching for: 

```
{
    "GNMI": {
        "certs":
        "gnmi": 
    }
    DEVICE_METADATA: {
        x509:
    }
}
```

I won't go though every line of [gnmi-native.sh](https://github.com/sonic-net/sonic-buildimage/blob/master/dockers/docker-sonic-gnmi/gnmi-native.sh) but you've got an idea on what this script is looking for and where. 

Based on it the expected configuration would be:

```
{
    "TELEMETRY": {
        "certs": {
            "ca_crt": "",
            "server_crt": "",
            "server_key": ""
        },
        "gnmi": {
            "client_auth": "",
            "log_level": "",
            "port": "",
            "threshold": "",
            "idle_conn_duration: "",
            "user_auth" : ""
        }
    }
}
```


Ultimately, the configuration is translated into `telemetry` process arguments and the process is lunched:

```
exec /usr/sbin/telemetry ${TELEMETRY_ARGS}
```

#### GNMI Configuration


To configure Telemetry and gNMI on SONiC, we need to push gNMI telelmetry items into the configuration database. A sample configuration database schema for gNMI and telemetry items is:

```
{
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
    }
}
```

Pushing this configuration to the router will start a gRPC server that will listen on port 50051 for gNMI client requests. 

Here the `certs` key corresponds to TLS certificates used by the telemetry infrastructure to ensure that traffic is streamed on a secure TLS-encrypted channel. Within this section, a user can specify where to find the certificates on the router.

The `gnmi` key allows users to configure gNMI parameters such as:

- `client_auth`: Whether the telemetry stream will be encrypted or not
- `log_level`: Set the log level
- `port`: The TCP port on which the gRPC server will listen for requests


#### Example configuration workflow

Create a `json` file containing configuration items.

```
cat telemetry.json

{
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
  }
}
```

Write this generated configuration to the configuration database

```
sonic-cfggen -j telemetry.json --write-to-db 
```

To ensure this configuration is persistent across reloads, issue a `config save` command

```
sudo config save -y
```

For troubleshooting, we can view the running configuration using the `show runningconfiguration all` command and filtering for `GNMI`.

For example:

```
show runningconfiguration all | grep GNMI -A 11


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

Additionally, we can view the startup configuration in `/etc/sonic/config_db.json`

```
jq '{GNMI: .GNMI}' /etc/sonic/config_db.json


{
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
  }
}
```

SONiC requires certificates to operate with the gNMI protocol. Certificates can be managed using a custom PKI infrastructure or self-signed certificates. 

In the next articles we'll go over certificates management examples. 


## Test tools installation

To test SONiC gNMI telemetry, we need to install gNMI client. 

Examples of gNMI clients are:

### pygnmi

A popular python based gNMI client. It can be installed using Python's `pip` package manager.

```
pip install pygnmi
```

### gNMIc

A CLI-integrated gNMI client. Installation instructions can be found in the following link:

<https://gnmic.openconfig.net/install/>


## gNMI capabilites

Here is a command example to get gNMI capabilities from c1 router using the previously created certificates. We assume the certificate are located in `./certs/` folder:


### pygnmicli example:

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 -o capabilities
```

Capabilities output example

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 -r "./certs/RootCA.crt" -c "./certs/c1/c1.sonic.cisco.com.cer" -k "./certs/c1/c1.sonic.cisco.com.key" -u cisco -p cisco123 -o capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing capabilities request to ('1.18.1.1', 50051)...
Collecting Capabilities...
Collection of Capabilities is successfull
{
    "supported_models": [
        {
            "name": "openconfig-acl",
            "organization": "OpenConfig working group",
            "version": "1.0.2"
        },
        {
            "name": "openconfig-interfaces",
            "organization": "OpenConfig working group",
            "version": ""
        },
        {
            "name": "openconfig-acl",
            "organization": "OpenConfig working group",
            "version": ""
        },
        {
            "name": "openconfig-sampling-sflow",
            "organization": "OpenConfig working group",
            "version": ""
        },
        {
            "name": "openconfig-lldp",
            "organization": "OpenConfig working group",
            "version": "1.0.2"
        },
        {
            "name": "openconfig-platform",
            "organization": "OpenConfig working group",
            "version": "1.0.2"
        },
        {
            "name": "openconfig-system",
            "organization": "OpenConfig working group",
            "version": "1.0.2"
        },
        {
            "name": "ietf-yang-library",
            "organization": "IETF NETCONF (Network Configuration) Working Group",
            "version": "2016-06-21"
        },
        {
            "name": "sonic-db",
            "organization": "SONiC",
            "version": "0.1.0"
        }
    ],
    "supported_encodings": [
        "json",
        "json_ietf",
        "proto"
    ],
    "gnmi_version": "0.7.0"
}
```

### gnmic example:

```
gnmic -a 1.18.1.1:50051 \
    --tls-ca "./certs/RootCA.crt" \
    --tls-cert "./certs/c1.sonic.cisco.com.crt" \
    --tls-key "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    capabilities
```

Capabilities output example

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


## Getting data from SONiC over gNMI examples

Below are some examples of getting different information from SONiC devices using gNMI:

### openconfig-interfaces:interfaces

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    -o get -x /openconfig-interfaces:interfaces
```

Output Example:

```
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731589404316392225,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "openconfig-interfaces:interfaces",
                    "val": {
                        "openconfig-interfaces:interfaces": {
                            "interface": [
                                {
                                    "config": {
                                        "enabled": true,
                                        "mtu": 9100,
                                        "name": "Ethernet0"
                                    },
                                    "openconfig-if-ethernet:ethernet": {
                                        "config": {
                                            "port-speed": "openconfig-if-ethernet:SPEED_400GB"
                                        },
                                        "state": {
                                            "counters": {
                                                "openconfig-if-ethernet-ext:in-distribution": {
                                                    "in-frames-1024-1518-octets": "0",
                                                    "in-frames-128-255-octets": "2",
                                                    "in-frames-256-511-octets": "21846",

            ...

```

### openconfig-interfaces:interfaces - Limiting output to Ethernet0 

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    -o get -x /openconfig-interfaces:interfaces/interface[name=Ethernet0]
```

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1/c1.sonic.cisco.com.cer" \
    -k "./certs/c1/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    -o get -x /openconfig-interfaces:interfaces/interface[name=Ethernet0]
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731589544400579753,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "openconfig-interfaces:interfaces/interface[name=Ethernet0]",
                    "val": {
                        "openconfig-interfaces:interface": [
                            {
                                "config": {
                                    "enabled": true,
                                    "mtu": 9100,
                                    "name": "Ethernet0"
                                },
                                "openconfig-if-ethernet:ethernet": {
                                    "config": {
                                        "port-speed": "openconfig-if-ethernet:SPEED_400GB"
                                    },
                                    "state": {
                                        "counters": {
                                            "openconfig-if-ethernet-ext:in-distribution": {
                                                "in-frames-1024-1518-octets": "0",
                                                "in-frames-128-255-octets": "2",
                                                "in-frames-256-511-octets": "21851",
                                                "in-frames-512-1023-octets": "0",
                                                "in-frames-64-octets": "0",
                                                "in-frames-65-127-octets": "0"
                                            },
                                            "in-oversize-frames": "0",
                                            "in-undersize-frames": "0"
                                        },
                                        "port-speed": "openconfig-if-ethernet:SPEED_400GB"
                                    }
                                },
                                "name": "Ethernet0",
                                "state": {
                                    "admin-status": "UP",
                                    "counters": {
                                        "in-broadcast-pkts": "0",
                                        "in-discards": "2",
                                        "in-errors": "0",
                                        "in-multicast-pkts": "21853",
                                        "in-octets": "5594124",
                                        "in-pkts": "21853",
                                        "in-unicast-pkts": "0",
                                        "out-broadcast-pkts": "0",
                                        "out-discards": "0",
                                        "out-errors": "0",
                                        "out-multicast-pkts": "0",
                                        "out-octets": "5703976",
                                        "out-pkts": "21857",
                                        "out-unicast-pkts": "21857"
                                    },
                                    "description": "",
                                    "enabled": true,
                                    "mtu": 9100,
                                    "name": "Ethernet0"
                                },
                                "subinterfaces": {
                                    "subinterface": [
                                        {
                                            "config": {
                                                "index": 0
                                            },
                                            "index": 0,
                                            "openconfig-if-ip:ipv6": {
                                                "config": {
                                                    "enabled": true
                                                },
                                                "state": {
                                                    "enabled": true
                                                }
                                            },
                                            "state": {
                                                "index": 0
                                            }
                                        }
                                    ]
                                }
                            }
                        ]
                    }
                }
            ]
        }
    ]
}
```

### openconfig-lldp:lldp

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    -o get -x /openconfig-lldp:lldp/interfaces/interface[name=Ethernet8]
```

output example

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1/c1.sonic.cisco.com.cer" \
    -k "./certs/c1/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    -o get -x /openconfig-lldp:lldp/interfaces/interface[name=Ethernet8]
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731589933866233231,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "openconfig-lldp:lldp/interfaces/interface[name=Ethernet8]",
                    "val": {
                        "openconfig-lldp:interface": [
                            {
                                "name": "Ethernet8",
                                "neighbors": {
                                    "neighbor": [
                                        {
                                            "capabilities": {
                                                "capability": [
                                                    {
                                                        "name": "openconfig-lldp-types:MAC_BRIDGE",
                                                        "state": {
                                                            "enabled": true,
                                                            "name": "openconfig-lldp-types:MAC_BRIDGE"
                                                        }
                                                    },
                                                    {
                                                        "name": "openconfig-lldp-types:ROUTER",
                                                        "state": {
                                                            "enabled": true,
                                                            "name": "openconfig-lldp-types:ROUTER"
                                                        }
                                                    }
                                                ]
                                            },
                                            "id": "Ethernet8",
                                            "state": {
                                                "chassis-id": "90:eb:50:a3:4b:68",
                                                "chassis-id-type": "MAC_ADDRESS",
                                                "id": "3",
                                                "management-address": "1.18.1.4,fd00::1",
                                                "port-description": "Ethernet16",
                                                "port-id": "etp2",
                                                "port-id-type": "LOCAL",
                                                "system-description": "SONiC Software Version: SONiC.202405c.16350-int-20240926.071054 - HwSku: 32x400Gb - Distribution: Debian 12.7 - Kernel: 6.1.0-11-2-amd64",
                                                "system-name": "c4"
                                            }
                                        }
                                    ]
                                }
                            }
                        ]
                    }
                }
            ]
        }
    ]
}
```


## Get Configuration from CONFIG_DB

###  PORT/Ethernet0

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target CONFIG_DB \
    -o get -x PORT/Ethernet0
```

Output example

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1/c1.sonic.cisco.com.cer" \
    -k "./certs/c1/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target CONFIG_DB \
    -o get -x PORT/Ethernet0
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731590130064395032,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "PORT/Ethernet0",
                    "val": {
                        "admin_status": "up",
                        "alias": "etp0",
                        "index": "0",
                        "lanes": "2304,2305,2306,2307,2308,2309,2310,2311",
                        "mtu": "9100",
                        "speed": "400000"
                    }
                }
            ]
        }
    ]
}
```


###  Telemetry configuration

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target CONFIG_DB \
    -o get -x GNMI
```

Output example:

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1/c1.sonic.cisco.com.cer" \
    -k "./certs/c1/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target CONFIG_DB \
    -o get -x GNMI
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731590289971146699,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "GNMI",
                    "val": {
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
                    }
                }
            ]
        }
    ]
}
```

## Data from COUNTERS_DB database

Counters for Ethernet0

```
pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1.sonic.cisco.com.crt" \
    -k "./certs/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target COUNTERS_DB \
    -o get -x COUNTERS/Ethernet0
```

Output example:

```
root@8ktme:/home/cisco# pygnmicli -t 1.18.1.1:50051 \
    -r "./certs/RootCA.crt" \
    -c "./certs/c1/c1.sonic.cisco.com.cer" \
    -k "./certs/c1/c1.sonic.cisco.com.key" \
    -u cisco -p cisco123 \
    --gnmi-path-target COUNTERS_DB \
    -o get -x COUNTERS/Ethernet0
Collecting Capabilities...
Collection of Capabilities is successfull
Selected encoding 'json' based on capabilities
Collecting Capabilities...
Collection of Capabilities is successfull
Doing get request to ('1.18.1.1', 50051)...
Collecting info from requested paths (Get operation)...
{
    "notification": [
        {
            "timestamp": 1731590441333983199,
            "prefix": null,
            "alias": null,
            "atomic": false,
            "update": [
                {
                    "path": "COUNTERS/Ethernet0",
                    "val": {
                        "SAI_PORT_STAT_ETHER_IN_PKTS_1024_TO_1518_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_128_TO_255_OCTETS": "2",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_1519_TO_2047_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_256_TO_511_OCTETS": "21881",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_512_TO_1023_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_64_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_IN_PKTS_65_TO_127_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_1024_TO_1518_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_128_TO_255_OCTETS": "4",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_1519_TO_2047_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_256_TO_511_OCTETS": "21882",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_512_TO_1023_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_64_OCTETS": "1",
                        "SAI_PORT_STAT_ETHER_OUT_PKTS_65_TO_127_OCTETS": "0",
                        "SAI_PORT_STAT_ETHER_RX_OVERSIZE_PKTS": "0",
                        "SAI_PORT_STAT_ETHER_STATS_TX_NO_ERRORS": "21887",
                        "SAI_PORT_STAT_ETHER_STATS_UNDERSIZE_PKTS": "0",
                        "SAI_PORT_STAT_ETHER_TX_OVERSIZE_PKTS": "0",
                        "SAI_PORT_STAT_IF_IN_BROADCAST_PKTS": "0",
                        "SAI_PORT_STAT_IF_IN_DISCARDS": "2",
                        "SAI_PORT_STAT_IF_IN_ERRORS": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0": "51285065114051",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S1": "433509717",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S10": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S11": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S12": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S13": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S14": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S15": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S2": "3632",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S3": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S4": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S5": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S6": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S7": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S8": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S9": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES": "433513349",
                        "SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES": "0",
                        "SAI_PORT_STAT_IF_IN_FEC_SYMBOL_ERRORS": "433516981",
                        "SAI_PORT_STAT_IF_IN_MULTICAST_PKTS": "21883",
                        "SAI_PORT_STAT_IF_IN_NON_UCAST_PKTS": "21883",
                        "SAI_PORT_STAT_IF_IN_OCTETS": "5601804",
                        "SAI_PORT_STAT_IF_IN_UCAST_PKTS": "0",
                        "SAI_PORT_STAT_IF_IN_UNKNOWN_PROTOS": "0",
                        "SAI_PORT_STAT_IF_OUT_BROADCAST_PKTS": "0",
                        "SAI_PORT_STAT_IF_OUT_DISCARDS": "0",
                        "SAI_PORT_STAT_IF_OUT_ERRORS": "0",
                        "SAI_PORT_STAT_IF_OUT_MULTICAST_PKTS": "0",
                        "SAI_PORT_STAT_IF_OUT_NON_UCAST_PKTS": "0",
                        "SAI_PORT_STAT_IF_OUT_OCTETS": "5711806",
                        "SAI_PORT_STAT_IF_OUT_UCAST_PKTS": "21887",
                        "SAI_PORT_STAT_IN_DROPPED_PKTS": "0",
                        "SAI_PORT_STAT_IP_IN_UCAST_PKTS": "0",
                        "SAI_PORT_STAT_OUT_DROPPED_PKTS": "0",
                        "SAI_PORT_STAT_PAUSE_RX_PKTS": "0",
                        "SAI_PORT_STAT_PAUSE_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_0_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_0_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_1_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_1_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_2_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_2_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_3_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_3_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_4_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_4_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_5_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_5_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_6_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_6_TX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_7_RX_PKTS": "0",
                        "SAI_PORT_STAT_PFC_7_TX_PKTS": "0"
                    }
                }
            ]
        }
    ]
}
```
