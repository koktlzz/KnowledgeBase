---
title: "Operator 自定义配置"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  prometheus:
    parent: "Prometheus"
weight: 200
---

通常，我们需要对 Prometheus 的监控、告警规则和消息推送进行一些自定义的配置。对于部署在虚拟机的 Prometheus 和 Alertmanager 实例来说，上述配置分别对应以下文件：

- `prometheus.yaml`
- `*-rules.yaml`
- `alertmanager.yaml`

但如果使用 Operator 在 Kubernetes 或 Openshift 中部署 Prometheus，其 Prometheus 和 Alertmanager 实例由对应的 Operator 负责管理，自然无法通过修改 Pod 挂载的 ConfigMap 或 Secret 来更新配置。因此，我们只能直接修改与其有关的 CRD（Custom Resource Definition）配置。

Prometheus Operator 在集群中创建的 CRD 资源主要有：

- *Prometheus*：管理集群中的 Prometheus StatefulSet 实例；
- *ServiceMonitor*：通过 Label Selector 选取需要监控的 Endpoint 对象；
- *Alertmanager*：管理集群中的 Alertmanager StatefulSet 实例；
- *PrometheusRule*：将告警规则配置动态加载到 Prometheus 实例中。

接下来我们将分别讨论两者之间的对应关系。为了区分 CRD 资源和原生的 Prometheus 概念，文中的 CRD 名称均以斜体表示。

## 监控

`prometheus.yaml`的示例文件如下，其中各字段的定义详见 [官方文档](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration)。

```yaml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  externalLabels:          # The labels to add to any time series or alerts when communicating with external systems (federation, remote storage, Alertmanager).
    cluster: my-k8s
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
```

与之对应的 *Prometheus* 对象配置则为：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: openshift-monitoring
spec:
  # global
  scrapeInterval: 15s
  evaluationInterval: 15s
  externalLabels:
    cluster: my-k8s
  # alerting
  alerting:
    alertmanagers:
    - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      name: alertmanager-main
      namespace: openshift-monitoring
      port: web
      scheme: https
      tlsConfig:
        ca: {}
        caFile: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt
        cert: {}
        serverName: alertmanager-main.openshift-monitoring.svc
  # rule_file
  ruleNamespaceSelector: {}
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  # scrape_configs
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
    matchLabels:
      k8s-app: node-exporter
```

其中，`global`下属的子字段（如`scrape_interval`、`evaluation_interval`和`externalLabels`）可以在 *Prometheus* 的`spec`中直接定义，不过写法有所差异。全部的字段对应关系可以查看 Prometheus Operator 官方给出的 [API 文档](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md#prometheusspec)。

值得注意的是，如果是在 Openshift 而非原生的 Kubernetes 集群中，使用`oc edit prometheus`来修改`spec`会在一段时间后回退为默认配置。我们需要在 openshift-monitoring 的 project 中创建一个名为 cluster-monitoring-config 的 ConfigMap，只有在其中定义全局配置才会生效：

```yaml
apiVersion: v1
data:
  config.yaml: |
    prometheusK8s:
      scrapeInterval: 15s
      evaluationInterval: 15s
      externalLabels:
        cluster: my-k8s
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
```

由于 Alertmanger 实例是由其对应的 CRD 对象管理的，因此`alerting`下属的子字段不再是静态配置（static_configs），而是 *Alertmanager* 对象及其证书信息。

`scrape_configs`和`rule_file`下属的配置则需要在与该 *Prometheus* 关联的 *ServiceMonitor* 和 *PrometheusRule* 对象中声明，它们分别由对应的 Selector 字段（`serviceMonitorNamespaceSelector`、`serviceMonitorSelector`、`ruleNamespaceSelector`和`ruleSelector`）所指定。Selector 字段值类型均为 Kubernetes 中的 *[metav1.LabelSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#labelselector-v1-meta)，因此一个空的 Label Selector 将选取所有对象。

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: openshift-monitoring
spec:
  endpoints:
  - bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    bearerTokenSecret:
      key: ""
    interval: 30s
    port: https
    relabelings:
    - action: replace
      regex: (.*)
      replacement: $1
      sourceLabels:
      - __meta_kubernetes_pod_node_name
      targetLabel: instance
    scheme: https
    tlsConfig:
      ca: {}
      caFile: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt
      cert: {}
      serverName: node-exporter.openshift-monitoring.svc
  jobLabel: k8s-app
  namespaceSelector: {}
  selector:
    matchLabels:
      k8s-app: node-exporter
```

该 *ServiceMonitor* 的 Label 为`k8s-app: node-exporter`，因此会被示例的 *Prometheus* 对象选取。*ServiceMonitor* 的详细字段定义同样可以在 [API 文档](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md#servicemonitorspec) 中查阅，此处简单介绍几个常用的字段：

- `endpoint.interval`：与全局的`scrapeInterval`相比，该字段只对选取的 Endpoint 对象有效；
- `endpoint.port`：Endpoint 暴露的 metrics 端口，即 Service 中的`targetPort`；
- `endpoint.relabeling`：等效于`prometheus.yaml`文件中的`relabel_config`字段；
- `jobLabel`：该字段值 k8s-app 将从`selector.matchLabels`中选取 node-exporter 作为数据的 job 标签，相当于在`prometheus.yaml`文件中指定`job_name`为node-exporter；
- `namespaceSelector`： 指定 Endpoint 对象所在的 Namespace，若设置`any: true`则将选取所有的 Namespace；
- `selector`：通过 Label Selector 选取需要监控的 Endpoint 对象。

对 *Prometheus* 和 *ServiceMonitor* 配置完毕后，我们可以在 Prometheus 容器中的`/etc/prometheus/config_out`目录下找到文件`prometheus.env.yaml`，它将代替`/etc/prometheus/prometheus.yaml`作为 Prometheus 实例的配置文件。

## 告警规则

告警规则的配置应在 *PrometheusRule* 对象中进行声明，其形式与`*-rules.yaml`完全相同：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: app-rules
  namespace: openshift-monitoring
spec:
  groups:
  - name: app-rules
    rules:
    - alert: APP_NotReady
      annotations:
        description: No any pod is ready for app {{ $labels.container}} .
        summary: App Not Ready
      expr: sum(kube_pod_container_status_ready{namespace="app",container != "deployment"})
        by (container) < 1
      for: 1m
      labels:
        severity: critical
        user: app-admin
```

该对象的 Label 为`prometheus: k8s`和`role: alert-rules`，因此会被示例的 *Prometheus* 对象选取。同样，我们可以在 Prometheus 容器中的`/etc/prometheus/rules`目录下找到对应的`*-rules.yaml`文件。

## 消息推送

告警消息推送的配置在 Alertmanager StatefulSet 挂载的 Secret 中，我们将其中的内容使用 Base64 解码：

```shell
[root@bastion ~]# oc -n openshift-monitoring get secret alertmanager-main --template='{{ index .data "alertmanager.yaml" }}' |base64 -d > alertmanager.yaml
[root@bastion ~]# cat alertmanager.yaml 
global:
  smtp_smarthost: "******"
  smtp_from: "prometheus@me.com"
route:
  group_by: [alertname]
  receiver: default
receivers:
  - name: default
    email_configs:
      - to: "app@me.com"
        send_resolved: true
```

因此，我们可以直接修改名为 alertmanager-main 的 Secret 来修改 Alertmanager 的告警消息推送配置。

## 参考文献

[Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration)

[Prometheus Operator API](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md)

[How to configure prometheus remote_write / remoteWrite in OpenShift Container Platform 4.x](https://access.redhat.com/solutions/4931911)

[Kubernetes API Reference Docs](https://v1-17.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#-strong-api-overview-strong-)
