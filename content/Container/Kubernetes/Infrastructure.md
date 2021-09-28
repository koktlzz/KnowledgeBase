---
title: "Infrastructure"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  container:
    parent: "Kubernetes"
weight: 800
---

## 架构

![202105062142](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/202105062142.jpeg)

## 基本特性

- 声明式：Kubernetes 中常用 yaml 文件定义服务和资源的拓扑结构和状态，它是一种声明式的编程方式，更加关注状态和结果。其实我们最常接触的是命令式编程，它要求我们描述为了达到某一个效果或者目标所需要完成的指令。常见的编程语言 Go、Ruby、C++ 以及我们使用的 kubectl 工具其实都属于命令式的编程方法；
- 显式接口：不存在内部的私有接口；
- 无侵入：每一个应用或者服务一旦被打包成了镜像就可以直接在 Kubernetes 中无缝使用，不需要修改应用程序中的任何代码；
- 可移植：支持有状态服务的迁移和持久化存储。

## 组件

### Master 组件

- etcd：保存了整个集群的状态；
- Api-server：负责处理来自用户的请求，其主要作用就是对外提供 RESTful 的接口。包括用于查看集群状态的读请求以及改变集群状态的写请求，也是唯一一个与 etcd 集群通信的组件；
- Controller-manager：运行了一系列的 Controller 进程，它们会按照用户的期望状态在后台不断地调节整个集群中的对象。当服务的状态发生了改变，控制器就会发现这个改变并且开始向目标状态迁移。每种资源都有其对应的 Controller；
- Scheduler：负责资源的调度，采用预算策略或优选策略将 Pod 调度到合适的节点上。

### Node 组件

- Kubelet：负责维持容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理。它周期性地从 API Server 获取 Pod 的 Spec，并确保容器处于运行状态且保持健康；
- Kube-proxy：负责为 Service 提供集群内部的服务发现和负载均衡；
- Container Runtime：负责运行容器的软件，如 Docker、CRI-O 以及所有实现 Kubernetes CRI（Container Runtime Interface）的应用。

### Client 组件

- kubeadm：部署工具，提供 **kubeadm init** 和 **kubeadm join**，用于快速部署 Kubernetes 集群；
- kubectl：用户或开发者使用的命令行工具。

### 插件 Addons

- CoreDNS：负责为整个集群提供 DNS 服务；
- Ingress Controller：为 Service 提供从集群外部访问的入口；
- Heapster：提供资源监控；
- Dashboard：提供 GUI；
- Federation：提供跨可用区的集群；
- Fluentd/Elasticsearch：提供集群日志采集、存储与查询；
- Prometheus：提供集群的监控能力。

## 资源与对象

> Kubernetes 中的所有内容都被抽象为“资源”，如 Pod、Service、Node 等都是资源。“对象”就是“资源”的实例，是持久化的实体。如某个具体的 Pod、某个具体的 Node。Kubernetes 使用这些实体去表示整个集群的状态。
> 对象的创建、删除、修改都是通过 Kubernetes API，也就是 Api Server 组件提供的 API 接口，这些是 RESTful 风格的 Api，与 k8s 的**万物皆对象**理念相符。命令行工具 kubectl，实际上也是调用 kubernetes api。
> Kubernetes REST API 中的所有对象都用 Name 和 UID 来明确地标识。

常见的资源配置信息有：

- apiVersion：api 版本
- kind：类别
- metadata：元数据，包含了资源的 Name, UID 和 Label 等
- spec：规格
- status：状态

使用 **kubectl get \<resource\> -o yaml** 命令即可在标准输出中看到某一种资源的 yaml 配置信息，如：

```yaml
- apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      openshift.io/deployment-config.latest-version: "2"
    labels:
      app: eureka-server
    name: eureka-server-2-asfdd
    uid: ********
  spec:
    # ...
  status:
    # ...
```

### Label

apiVersion, kind 和 metadata 是每个资源和对象都拥有的字段，而 metadata 中除了规定了每个对象所独有的 Name 和 UID 外，还声明了对象的 Label，可以作为过滤项帮助我们选择和管理对象。

> *标签（Labels）* 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

如一个对象的 Label 配置信息如下：

```yaml
kind: Pod 
metadata: 
  labels: 
    key1 : value1,
    key2 : value2
```

我们就可以用命令 **kubectl get pods -l key1=value1,key2=value2** 准确的找到该对象。
给对象设置标签后，还可以在 yaml 文件中定义标签选择器过滤指定的标签。许多资源支持内嵌的标签选择器字段，如 matchLabels 和 matchExpressions。一个常见的使用场景是指定某一 Pod 的节点选择准则，方便节点调度。例如让一个 Pod 选择标签为 accelerator 为"nvidia-tesla-p100"的节点：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

还可以通过 Selector 指定 Service 指向的一组 Pod，以及 ReplicationController 应该管理的 Pods 的数量等。

### Spec 和 Status

- Spec 规定了对象的期望状态，不同对象的 Spec 几乎完全不同。
- Status 描述了对象的当前状态，是我们观察集群本身的一个接口。

> 在任何时刻，Kubernetes 都一直积极地管理着对象的实际状态 Status，以使之与期望状态 Spec 相匹配。
