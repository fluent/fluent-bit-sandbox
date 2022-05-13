---
slug: run-from-command-line
id: 3tfozpbz33wj
type: challenge
title: Fluent Bit collecting system metrics
teaser: Learn how to collect system metrics with Fluent Bit
notes:
- type: text
  contents: Please wait while we set up the first challenge
tabs:
- title: Terminal
  type: terminal
  hostname: fluent-bit-sandbox
- title: grafana
  type: service
  hostname: grafana
  port: 3000
- title: prometheus
  type: service
  hostname: prometheus
  port: 9090
- title: test
  type: terminal
  hostname: grafana
difficulty: basic
timelimit: 600
---
Introduction
==============

In this series we will learn how Fluent Bit integrates with systems such as Prometheus. Let's familarize ourself with the environment.

You will see a number of tabs on the left hand side, we have setup Fluent Bit, Prometheus, and Grafana a common stack used for metrics.

Throughout these tracks we will configure Fluent Bit to collect specific metrics, send those to Prometheus, and then visualize them with Grafana.


Running Fluent Bit via command line
==============
Let's first start by configuring Fluent Bit with a command line to see how metrics appear when collected. let's run the following

```
fluent-bit -i node_exporter_metrics -o stdout
```

The following command will use the [node_exporter_metrics](https://docs.fluentbit.io/manual/pipeline/inputs/node-exporter-metrics) plugin to collect Prometheus format data and then output to stdout


> fluent-bit collects node metrics by using the node_exporter_metrics plugin which mimics the Prometheus node exporter tool (No Prometheus needed)


Exiting Fluent Bit
==============
After you have run for a few seconds you can hit the following to exit

```
Ctrl - C
```

To complete this track click the **Check** button.
