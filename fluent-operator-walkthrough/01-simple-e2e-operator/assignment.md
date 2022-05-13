---
slug: simple-e2e-operator
id: xlpivpuzcv5u
type: challenge
title: Simple introduction using Fluent Bit to send logs to Grafana
teaser: Set up Fluent Bit to send your logs to Grafana
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes-vm
- title: Editor
  type: code
  hostname: kubernetes-vm
  path: /root/fluent-operator-walkthrough
- title: Grafana
  type: service
  hostname: kubernetes-vm
  port: 30001
  new_window: true
difficulty: basic
timelimit: 3600
---
Introduction to Fluent Operator
===============================

A simple end-to-end deployment of Fluent Bit forwarding to Loki and visible in Grafana all using the Fluent Operator.

This is based upon the examples from this repo: https://github.com/kubesphere-sigs/fluent-operator-walkthrough

Loki and Grafana
===================

This deploys the data sinks you want for your test cluster.

The repository should be already cloned to `~/fluent-operator-walkthrough` so:
```shell
cd ~/fluent-operator-walkthrough
./deploy-loki.sh
```
This is helper script that just uses the Grafana Helm chart to do the setup: https://github.com/kubesphere-sigs/fluent-operator-walkthrough/blob/master/deploy-loki.sh

Once this has finished deploying we can set up a service for it to be accessible externally for this environment:

```shell
kubectl apply -f ./grafana-service.yaml
```
Grafana should now be visible in the separate tab.

Make sure to get the `admin` user credentials:
```shell
kubectl get secret --namespace loki loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

The repository also includes examples for other endpoints, e.g. Kafka and Elasticsearch.
For other development tooling refer to this repository or the https://github.com/calyptia/fluent-bit-devtools repository as well.

Fluent Operator
===============

Now it is time to deploy the Fluent Operator, again via a helper script that invokes the Helm chart: https://github.com/kubesphere-sigs/fluent-operator-walkthrough/blob/master/deploy-fluent-operator.sh

```shell
cd ~/fluent-operator-walkthrough
./deploy-fluent-operator.sh
```

You can find more details of the Fluent Bit and Fluentd CRDs in the links below:

https://github.com/fluent/fluent-operator#fluent-bit
https://github.com/fluent/fluent-operator#fluentd

That's it, your Fluent Operator pods should now be running in the `fluent` namespace:
```shell
kubectl get pods -n fluent
```

Fluent Bit to Loki
==================
We now want to send all our logs to Loki using Fluent Bit.
To do this we need to set up a Custom Resource Definition (CRD) for the operator to pick up and manage.

We have split this into multiple YAML files: one to cover a Fluent Bit configuration to pick up all log and add various filters, etc. and then others to handle configuring a specific output to then send the logs too.

```shell
kubectl apply -f ~/fluent-operator-walkthrough/fluent-bit-inputs.yaml
kubectl apply -f ~/fluent-operator-walkthrough/fluent-bit-output-loki.yaml
```

We should see the FluentBit DaemonSet up and running after this with the various Custom Resources (CRs) as well showing configuration details.

```shell
kubectl -n fluent get daemonset
kubectl -n fluent get fluentbit
kubectl -n fluent get clusterfluentbitconfig
kubectl -n fluent get clusterinput.fluentbit.fluent.io
kubectl -n fluent get clusterfilter.fluentbit.fluent.io
kubectl -n fluent get clusteroutput.fluentbit.fluent.io
```

Within a couple of minutes, you can double check the output in the Grafana (or your choice of tooling).