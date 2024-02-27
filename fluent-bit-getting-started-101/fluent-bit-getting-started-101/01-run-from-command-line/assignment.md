---
slug: run-from-command-line
id: 1fukzww3cndt
type: challenge
title: Running Fluent Bit via command line
teaser: Learn how to run Fluent Bit via command line
notes:
- type: text
  contents: Please wait while we set up the first challenge
tabs:
- title: Terminal
  type: terminal
  hostname: fluent-bit-sandbox
difficulty: basic
timelimit: 600
---
Introduction
==============

In this series we will learn how to run Fluent Bit with the command line, and collect Host metrics like CPU, Disk, etc and print them out to stdout

In our environment we have `fluent-bit` installed which is a packaged version of Fluent Bit available across multiple operating systems and container environments.

> fluent-bit collects node metrics by using the node_exporter_metrics plugin which mimics the Prometheus node exporter tool (No Prometheus needed)

Running Fluent Bit via command line
==============
To run Fluent Bit with a command line let's run the following

```
fluent-bit -i node_exporter_metrics -o stdout
```
The following command will use the [node_exporter_metrics](https://docs.fluentbit.io/manual/pipeline/inputs/node-exporter-metrics) plugin to collect Prometheus format data and then output to stdout


Exiting Fluent Bit
==============
After you have run for a few seconds you can hit the following to exit

```
Ctrl - C
```

To complete this track click the **Check** button.
