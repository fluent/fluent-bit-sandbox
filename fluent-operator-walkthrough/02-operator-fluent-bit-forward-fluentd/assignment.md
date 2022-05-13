---
slug: operator-fluent-bit-forward-fluentd
id: 94ofvedo5mbb
type: challenge
title: Fluent Bit to Fluentd aggregation and control
teaser: Set up log forwarding to Fluentd from Fluent Bit and control outputs at cluster
  or namespace levels
tabs:
- title: Terminal
  type: terminal
  hostname: kubernetes-vm
- title: Editor
  type: code
  hostname: kubernetes-vm
  path: /root/fluent-operator-walkthrough
- title: Kibana
  type: service
  hostname: kubernetes-vm
  port: 30002
  new_window: true
- title: Grafana
  type: service
  hostname: kubernetes-vm
  port: 30001
  new_window: true
difficulty: basic
timelimit: 1200
---
Fluent Bit + Fluentd mode
=========================

With its rich plugins, Fluentd acts as a log aggregation layer and is able to perform more advanced log processing.

You can forward logs from Fluent Bit to Fluentd with ease using the Fluent Operator.
You can also control outputs from specific namespaces or the whole cluster being sent to different endpoints, particularly useful in a multi-tenant scenario.

Finally we will show custom routing to kafka by namespace and some specific buffering options in Fluentd.

Pre-requisites
==============

Ensure you have run the previous challenge or follow the pre-requisites defined in the Git repository: https://github.com/kubesphere-sigs/fluent-operator-walkthrough#prerequisites

Specifically we need to deploy a data sink (Kafka, Elasticsearch or Loki) as well as the Fluent operator:
```shell
cd ~/fluent-operator-walkthrough
./deploy-es.sh
./deploy-kafka.sh
./deploy-fluent-operator.sh
```

We also need to ensure Fluent Bit is running to pick up logs:
```shell
kubectl apply -f ~/fluent-operator-walkthrough/fluent-bit-inputs.yaml
```

See the Git repo for details on how to access logs, etc. from the destinations.

If you deploy Kibana then also use the service file to make it accessible from the tab above (same if you use Grafana which is done in the first challenge):

```shell
kubectl apply -f ./kibana-service.yaml
kubectl apply -f ./grafana-service.yaml
```

Set up forwarding from Fluent Bit to Fluentd
============================================

To forward logs from Fluent Bit to Fluentd, you need to enable the Fluent Bit forward plugin as below:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentbit.fluent.io/v1alpha2
kind: ClusterOutput
metadata:
  name: fluentd
  labels:
    fluentbit.fluent.io/enabled: "true"
    fluentbit.fluent.io/mode: "k8s"
spec:
  matchRegex: (?:kube|service)\.(.*)
  forward:
    host: fluentd.fluent.svc
    port: 24224
EOF
```

Enable Fluentd Forward Input plugin to receive logs from Fluent Bit
===================================================================

Deploy Fluentd with the forward input plugin:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: Fluentd
metadata:
  name: fluentd
  namespace: fluent
  labels:
    app.kubernetes.io/name: fluentd
spec:
  globalInputs:
  - forward:
      bind: 0.0.0.0
      port: 24224
  replicas: 1
  image: kubesphere/fluentd:v1.14.4
  fluentdCfgSelector:
    matchLabels:
      config.fluentd.fluent.io/enabled: "true"
EOF
```

ClusterFluentdConfig: Fluentd cluster-wide configuration
========================================================

You can send logs Fluentd received from Fluent Bit to a cluster-wide sink defined by a `ClusterOutput`.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - default
  - kafka
  - elastic
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/scope: "cluster"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-es
  labels:
    output.fluentd.fluent.io/scope: "cluster"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-fd
EOF
```

You should see the Fluentd StatefulSet come up and running with other CRs:

```shell
kubectl -n fluent get statefulset
kubectl -n fluent get fluentd
kubectl -n fluent get clusterfluentdconfig.fluentd.fluent.io
kubectl -n fluent get clusteroutput.fluentd.fluent.io
```

Wait a few minutes and then you should be able to query ES for the logs:
```shell
ubectl -n elastic exec -it elasticsearch-master-0 -c elasticsearch --  curl -X GET "localhost:9200/fluent-log*/_search?pretty" -H 'Content-Type: application/json' -d '{
   "size" : 0,
   "aggs" : {
      "kubernetes_ns": {
         "terms" : {
           "field": "kubernetes.namespace_name.keyword"
         }
      }
   }
}'
```
Output like the following should appear:
```yaml
{
  "took" : 203,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 67,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "kubernetes_ns" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "kafka",
          "doc_count" : 46
        },
        {
          "key" : "elastic",
          "doc_count" : 18
        },
        {
          "key" : "kube-system",
          "doc_count" : 3
        }
      ]
    }
  }
}
```
FluentdConfig: Fluentd namespace-only configuration
====================================================

You can send logs Fluentd has received from Fluent Bit to a namespace-wide sink defined by a `Output`.
Only logs in the same namespace as the `FluentdConfig` CR will be sent to the `Output`.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: namespace-fluentd-config
  namespace: kube-system
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  outputSelector:
    matchLabels:
      output.fluentd.fluent.io/scope: "namespace"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Output
metadata:
  name: namespace-fluentd-output-es
  namespace: kube-system
  labels:
    output.fluentd.fluent.io/scope: "namespace"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-namespace-fd
EOF
```

You should see the Fluentd CRs created:

```shell
kubectl -n fluent get fluentdconfig
kubectl -n fluent get output.fluentd.fluent.io
```

Mixing cluster and namespace logging output for Fluentd
===========================================================

You can use `FluentdConfig` and `ClusterFluentdConfig` together like below.
The `FluentdConfig` will only send logs from the `fluent` namespace pods to the `ClusterOutput`
The `ClusterFluentdConfig` will send logs of pods from all the `watchedNamespaces` to the `ClusterOutput`

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config-hybrid
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - default
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/scope: "hybrid"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: namespace-fluentd-config-hybrid
  namespace: fluent
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/scope: "hybrid"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-es-hybrid
  labels:
    output.fluentd.fluent.io/scope: "hybrid"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-hybrid-fd
EOF
```

You should see the Fluentd CRs created:

```shell
kubectl -n fluent get clusterfluentdconfig
kubectl -n fluent get fluentdconfig
kubectl -n fluent get clusteroutput.fluentd.fluent.io
```

Multi-tenant logging
====================

It may be useful in multi-tenant scenarios to mix-and-match the use of cluster logging with namespace logging.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: FluentdConfig
metadata:
  name: namespace-fluentd-config-user1
  namespace: fluent
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  outputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/user: "user1"
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/user: "user1"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config-cluster-only
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - kubesphere-system
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/enabled: "true"
      output.fluentd.fluent.io/scope: "cluster-only"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: Output
metadata:
  name: namespace-fluentd-output-user1
  namespace: fluent
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/user: "user1"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-user1-fd
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-user1
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/user: "user1"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-user1-fd
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-cluster-only
  labels:
    output.fluentd.fluent.io/enabled: "true"
    output.fluentd.fluent.io/scope: "cluster-only"
spec:
  outputs:
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-cluster-only-fd
EOF
```

You should see the Fluentd CRs created:

```shell
kubectl -n fluent get clusterfluentdconfig
kubectl -n fluent get fluentdconfig
kubectl -n fluent get output.fluentd.fluent.io
kubectl -n fluent get clusteroutput.fluentd.fluent.io
```

Route logs by namespace to different Kafka topics
=================================================

You can use a Fluentd filter to distribute logs to different topics based on namespace.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config-kafka
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - default
  clusterFilterSelector:
    matchLabels:
      filter.fluentd.fluent.io/type: "k8s"
      filter.fluentd.fluent.io/enabled: "true"
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/type: "kafka"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFilter
metadata:
  name: cluster-fluentd-filter-k8s
  labels:
    filter.fluentd.fluent.io/type: "k8s"
    filter.fluentd.fluent.io/enabled: "true"
spec:
  filters:
  - recordTransformer:
      enableRuby: true
      records:
      - key: kubernetes_ns
        value: ${record["kubernetes"]["namespace_name"]}
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-kafka
  labels:
    output.fluentd.fluent.io/type: "kafka"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - kafka:
      brokers: my-cluster-kafka-brokers.kafka.svc:9092
      useEventTime: true
      topicKey: kubernetes_ns
EOF
```

Fluentd output buffering
========================

You can add a buffer to cache logs for the output plugins.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFluentdConfig
metadata:
  name: cluster-fluentd-config-buffer
  labels:
    config.fluentd.fluent.io/enabled: "true"
spec:
  watchedNamespaces:
  - kube-system
  - default
  clusterFilterSelector:
    matchLabels:
      filter.fluentd.fluent.io/type: "buffer"
      filter.fluentd.fluent.io/enabled: "true"
  clusterOutputSelector:
    matchLabels:
      output.fluentd.fluent.io/type: "buffer"
      output.fluentd.fluent.io/enabled: "true"
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterFilter
metadata:
  name: cluster-fluentd-filter-buffer
  labels:
    filter.fluentd.fluent.io/type: "buffer"
    filter.fluentd.fluent.io/enabled: "true"
spec:
  filters:
  - recordTransformer:
      enableRuby: true
      records:
      - key: kubernetes_ns
        value: ${record["kubernetes"]["namespace_name"]}
---
apiVersion: fluentd.fluent.io/v1alpha1
kind: ClusterOutput
metadata:
  name: cluster-fluentd-output-buffer
  labels:
    output.fluentd.fluent.io/type: "buffer"
    output.fluentd.fluent.io/enabled: "true"
spec:
  outputs:
  - stdout: {}
    buffer:
      type: file
      path: /buffers/stdout.log
  - elasticsearch:
      host: elasticsearch-master.elastic.svc
      port: 9200
      logstashFormat: true
      logstashPrefix: fluent-log-buffer-fd
    buffer:
      type: file
      path: /buffers/es.log
EOF
```
