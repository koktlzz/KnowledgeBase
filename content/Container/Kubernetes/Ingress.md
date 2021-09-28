---
title: "Ingress"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  container:
    parent: "Kubernetes"
weight: 1300
---

## 概述

在集群外部，我们可以通过 [NodePort 类型的 Service](/kubernetes/kubernetes基础/service/#nodeport) 或 [LoadBalancer 类型的 Service](/kubernetes/kubernetes基础/service/#loadbalancer) 来访问集群内部的 Pod。由于它们的实现均基于 IP 地址和端口并通常使用 TCP 协议，因此属于四层网络模型。

Ingress 则基于七层网络模型，可以向集群外部提供可访问的 URL 并将发往 Ingress 负载均衡器的外部 HTTP 或 HTTPS 请求转发到集群内部的 Service。一个将所有流量都发送到同一 Service 下 Pod 的简单 Ingress 示意图如下：

![demo.001.jpeg.001](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/demo.001.jpeg.001.jpeg)

如图所示，集群外的客户端向 Ingress 负载均衡器发送 HTTP(s) 请求，流量将被转发到节点的 80(443) 端口上。同时，集群内部的 Ingress 对象监听节点的 80(443) 端口，因此流量将被其接收。Ingress 对象将请求的 URL 和目标主机名与其配置的规则进行匹配，并将流量转发到对应的 Service。最后由 Service 将流量转发到后端的 Pod。

在 Openshift 中，提供 Ingress 功能的组件被称为 Router。它以 Pod 的形式部署在集群的 Infra 或 Router 节点上，通过 [hostnetwork](https://rcarrata.com/openshift/ocp4_shard_same_node/#:~:text=This%20is%20because%20HostNetwork%20allowed%20the%20OpenShift%20Router,Balancing%20for%20an%20external%20physical%20or%20virtual%20LB.) 对外暴露 80（443）端口，其内部则运行着 HAProxy 服务。Router 会从 etcd 中查询 Route 对象绑定的 Service 的后端 Pod Endpoint 信息，并将其写入到 HAProxy 的路由规则中。这样当 Router 接收到发往其所在节点 80（443）端口的流量时，HAProxy 就可以直接将其转发到对应的 Pod 而无需经过 Service。

通常由 Ingress Controller 负责创建 Ingress 负载均衡器，因此在创建 Ingress 对象之前，集群中必须拥有 Ingress Controller。Kubernetes 目前支持和维护 [AWS](https://github.com/kubernetes-sigs/aws-load-balancer-controller#readme)，[GCE](https://git.k8s.io/ingress-gce/README.md) 和 [nginx](https://git.k8s.io/ingress-nginx/README.md#readme) 三种 Ingress Controller。

## 配置

一个最简单的 Ingress 对象配置如下：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress                                # 必须是合法的 DNS 子域名名称
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /      # 注解的配置取决于 Ingress Controller
spec:
rules:
- http:
    paths:
    - path: /testpath
    pathType: Prefix
    backend:
        service:
        name: test
        port:
            number: 80
```

Ingress 对象的`spec.rules`字段中规定了所有传入请求（incoming request）应匹配的规则列表，且仅支持转发 HTTP 或 HTTPS 流量的规则。每个 HTTP(s) 规则都应包含以下信息：

- `host`：可使用通配符来指定请求的目标主机名格式。若未指定，则该规则适用于指向 Ingress IP 地址的所有入站 HTTP 流量；
- `paths`：路径列表，其中的每个路径都有一个与之关联的后端（`backend`）；
- `pathType`：每个路径必须指定其类型，Kubernetes 中支持的类型有以下三种：
  - `ImplementationSpecific`：匹配规则取决于 Ingress Class，可以将其视为单独的`pathType`或与`Prefix`或`Exact`路径类型相同。
  - `Exact`：精确匹配 URL 路径，且区分大小写；
  - `Prefix`：根据以/分隔的 URL 路径前缀进行匹配，且区分大小写。如`path`字段值为`/aaa/bbb`，则请求路径`/aaa/bbb/ccc`与之匹配，而`/aaa/bbbccc`不匹配。
- `backend`：由`service.name`和`service.port.number`（或`service.port.name`）定义的 Service 对象，只有从与规则匹配的`host`和`path`发出的入站请求才会被 Ingress 转发到后段的 Service 中。通常在 Ingress Controller 中会配置`defaultBackend`（默认后端）字段，以处理任何不匹配规则列表的传入请求。

另外，`backend`还可以是一个对象引用（ObjectRef），指向 Ingress 对象所处 Namespace 下的另一个 Kubernetes 资源。此时配置中的字段变为`backend.resource`，其常见用法是将入站数据通过 Ingress 导入到存放静态资产（static assets）的对象存储中：

```yaml
...
rules:
- http:
    paths:
        - path: /icons
        pathType: ImplementationSpecific
        backend:
            resource:                       # resource 与 service 的配置互斥，若二者均被设置会无法通过检查
            apiGroup: k8s.example.com
            kind: StorageBucket
            name: icon-assets
```

## Ingress Class

Ingress 资源可以由不同的 Ingress Controller 创建，通常使用不同的配置。因此每个 Ingress 对象通过引用一个 Ingress Class，来指定 Ingress Controller 的种类和配置。Ingress、Ingress Class 和 Ingress Controller 三者之间的关系类似于 Volume 中的 PV、Storage Class 和 Storage Class Provisioner。

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller    # 实现 Ingress Class 的 Controller 名称
  parameters:                                   # 可选字段，为 Ingress Class 引用的额外配置
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

## 根据请求的 URI 路由

配置扇出（Fanout）可以根据 HTTP(S) 请求的 URI，将来自同一 IP 地址的流量路由到多个后端 Service。我们可以配置以下 Ingress 完成这一效果：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

查看已创建的 Ingress 配置，可以在`Address`字段中看到 Ingress 负载均衡器的 IP 地址：

```bash
kubectl describe ingress simple-fanout-example

Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

该 Ingress 处理入站请求的示意图如下：

![195698510203044522](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/195698510203044522.jpg)

## 根据请求的目标主机名路由

配置 Ingress 实现将指向同一 IP 地址、不同主机名的 HTTP(S) 请求的流量路由到不同的多个后端 Service，不过前提条件是已将目标主机域名解析为 Ingress 的 IP 地址。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress-no-third-host
spec:
  rules:
  - host: first.bar.com                        # 指向主机名 first.bar.com 的请求将被转发到 service1
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: second.bar.com                       # 指向主机名 second.bar.com 的请求将被转发到 service2
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service3                     # 未指定主机名但指向 Ingress IP 的请求将被转发到 servic3
            port:
              number: 80
```

该 Ingress 处理入站请求的示意图如下：

![235255507323325313](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/235255507323325313.jpg)

## 总结

至此，Kubernetes 中的网络模型已介绍完毕，下图对此进行了一个很好的总结：

![20210313164743](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210313164743.png)
