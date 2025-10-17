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
