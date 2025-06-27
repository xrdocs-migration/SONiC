---
published: false
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


Here we’ll focus on gNMI as our exampl but the approach we’re about to discuss is not limited to gNMI, it can be applied to any other SONiC SONiC.

