---
slug: operator-fluentd-only
id: 8b55iszvqqsm
type: challenge
title: Fluentd only example
teaser: Set up Fluentd to handle HTTP input
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes-vm
- title: Editor
  type: code
  hostname: kubernetes-vm
  path: /root/fluent-operator-walkthrough
- title: Terminal
  type: terminal
  hostname: kubernetes-vm
difficulty: basic
timelimit: 1200
---
Fluentd only mode
=================

This example shows how to set up Fluentd only with the operator, i.e. no forwarding from Fluent Bit, and receive logs from HTTP input.
The logs are only sent to standard output but you could use any of the other examples to send to Elasticsearch, Kafka, etc.

Fluent Operator
==============
If you have not run the other challenges then we need to deploy the Fluent Operator,  via a helper script that invokes the Helm chart: https://github.com/kubesphere-sigs/fluent-operator-walkthrough/blob/master/deploy-fluent-operator.sh

```shell
cd ~/fluent-operator-walkthrough
./deploy-fluent-operator.sh
```

Your Fluent Operator pods should now be running in the fluent namespace:

```shell
kubectl get pods -n fluent
```

Handle HTTP input
=================

You can use Fluentd only to receive logs via HTTP.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: Fluentd
metadata:
  name: fluentd-http
  namespace: fluent
  labels:
    app.kubernetes.io/name: fluentd
spec:
  globalInputs:
    - http:
        bind: 0.0.0.0
        port: 9880
  replicas: 1
  image: kubesphere/fluentd:v1.14.4
  fluentdCfgSelector:
    matchLabels:
      config.fluentd.fluent.io/enabled: "true"

---
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: fluentd-only-config
  namespace: fluent
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  filterSelector:
    matchLabels:
      filter.fluentd.fluent.io/mode: "fluentd-only"
      filter.fluentd.fluent.io/enabled: "true"
  outputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/mode: "fluentd-only"

---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Filter
metadata:
  name: fluentd-only-filter
  namespace: fluent
  labels:
    filter.fluentd.fluent.io/mode: "fluentd-only"
    filter.fluentd.fluent.io/enabled: "true"
spec:
  filters:
    - stdout: {}

---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Output
metadata:
  name: fluentd-only-stdout
  namespace: fluent
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/mode: "fluentd-only"
spec:
  outputs:
    - stdout: {}
EOF
```

You should see the Fluentd CRs created:

```shell
kubectl -n fluent get fluentd
kubectl -n fluent get fluentdconfig
kubectl -n fluent get filter.fluentd.fluent.io
kubectl -n fluent get output.fluentd.fluent.io
```

Then you can send logs to the endpoint by using curl:

```shell
curl -X POST -d 'json={"foo":"bar"}' http://<localhost:9880>/app.log
```
> replace the <localhost:9880> with the actual service or endpoint of fluentd, i.e: fluentd-http.fluent.svc:9880

With a separate terminal you can tail the logs of the Fluentd pod:

```shell
kubectl -n fluent logs -f fluentd-0
```

View the Fluentd configuration
==============================

```shell
kubectl -n fluent get secrets fluentd-config -ojson | jq '.data."app.conf"' | awk -F '"' '{printf $2}' | base64 --decode
---
<source>
  @type  forward
  bind  0.0.0.0
  port  24224
</source>
<match **>
  @id  main
  @type  label_router
  <route>
    @label  @48b7cb809bc2361ba336802a95eca0d4
    <match>
      namespaces  kube-system,default
    </match>
  </route>
</match>
<label @48b7cb809bc2361ba336802a95eca0d4>
  <filter **>
    @id  ClusterFluentdConfig-cluster-fluentd-config::cluster::clusterfilter::fluentd-filter-0
    @type  record_transformer
    enable_ruby  true
    <record>
      kubernetes_ns  ${record["kubernetes"]["namespace_name"]}
    </record>
  </filter>
  <match **>
    @id  ClusterFluentdConfig-cluster-fluentd-config::cluster::clusteroutput::fluentd-output-es-0
    @type  elasticsearch
    host  elasticsearch-master.elastic.svc
    logstash_format  true
    logstash_prefix  fluent-log
    port  9200
  </match>
  <match **>
    @id  ClusterFluentdConfig-cluster-fluentd-config::cluster::clusteroutput::fluentd-output-kafka-0
    @type  kafka2
    brokers  my-cluster-kafka-brokers.kafka.svc:9092
    topic_key  kubernetes_ns
    use_event_time  true
    <format>
      @type  json
    </format>
  </match>
</label>
```