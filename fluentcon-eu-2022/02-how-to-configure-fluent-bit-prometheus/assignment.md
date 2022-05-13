---
slug: how-to-configure-fluent-bit-prometheus
id: cc2o1mcpepm0
type: challenge
title: Using Fluent Bit to send metrics to Prometheus, via remote write
teaser: In this challenge learn how to use Fluent Bit to collect metrics and send
  data to Prometheus via remote write
tabs:
- title: Terminal
  type: terminal
  hostname: fluent-bit-sandbox
- title: Fluent Bit Configuration
  type: code
  hostname: fluent-bit-sandbox
  path: /fluent-bit/etc/
- title: Fluent Bit Documentation
  type: website
  hostname: fluent-bit
  url: https://docs.fluentbit.io/manual/
- title: Grafana
  type: service
  hostname: grafana
  port: 3000
- title: Prometheus
  type: service
  hostname: prometheus
  port: 9090
difficulty: basic
timelimit: 600
---
🤖 Introduction
==============

In the last lesson we walked through how to collect metrics with Fluent Bit from a command line. When running in production however we will can't watch the terminal for metric changes and we will need to visualize these metrics.

To do this, we will configure Fluent Bit to use the Prometheus Remote Write output plugin to route the data collected by node_exporter_plugin to Prometheus via the remote write protocol. We will then use Grafana to then visualize these metrics.


Accessing the configuration
==============

You will notice a new tab (Fluent Bit Configuration) that is now shown next to your terminal window. This points to the `/fluent-bit/etc/` directory. Additionally, you have the official Fluent Bit documentation in a tab window as well (Fluent Bit Documentation)

Within this directory you will find the `/fluent-bit/etc/fluent-bit.conf` file that you can modify. This directory also contains the default parsers file `parsers.conf` that contains out of the box parsers like docker, json, apache2, cri, and more.

> Note: When modifying the configuration in the Fluent Bit Configuration tab you will need to click the save icon on the tab to save changes

Modifying the configuration
===============
Let's add a new INPUT plugin to read the node_exporter_plugin data. To do this under the existing INPUT section we will add the following code block

```
[INPUT]
    name node_exporter_plugin
    tag metrics

[OUTPUT]
    name Prometheus_remote_write
    match metrics
    host prometheus
    port 9200
```

Running Fluent Bit
=================
Let's run Fluent Bit the same way as we did before and this time use the parameter `-c` to indicate a configuration file.

```
fluent-bit -c /fluent-bit/etc/fluent-bit.conf
```
Visualizing data with Grafana
=================
Now that data is hopefully flowing to Prometheus, let's switch to the Grafana tab and go to `Dashboards` and `Node Exporter Dashboards`. After a few seconds we should see data load up in the dashboard.
