---
title: "Service"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  container:
    parent: "Kubernetes"
weight: 1000
---

## 概述

- Kubernetes 使用 Service 解决服务发现问题：每个 Pod 在创建后都会被分配一个 IP 地址，然而它会随着 Pod 的重启而改变；
- Service 可以通过标签选择器选择一组 Pod，然后作为它们共同的对外访问接口。这样我们的应用便可以在不知道 Pod 的 IP 地址的情况下，与其通信；
- 当 Service 的标签选择器选择了多个 Pod 时，还可以在它们之间做负载均衡；
- 众所周知，Service 的中文意为“服务”。但就其功能而言，更像是一个 Proxy（代理）或 Router（路由）。

## 配置

一个典型的 Service 对象配置如下：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-server
spec:
  clusterIP: 192.168.1.0
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

- 每个 Service 都会由系统分配一个虚拟 IP 作为访问 Service 的入口 IP 地址，然后监听`spec.ports.port`字段指定的端口；
- Service 的 IP 也可以通过`spec.clusterIP`字段指定，且必须是 APIServer 中的配置字段`service-cluster-ip-range` CIDR 范围内的合法地址；
- 标签选择器`spec.selector`能够根据 labels 选择目标 Pod，而 Service 会将外部流量转发到目标 Pod 的`spec.ports.targetPort`端口；
- 该 Service 开放了多个端口（80/443），因此必须定义端口的名称`spec.ports.name`（http/https），以避免歧义；
- 当 Service 被创建后，系统随之创建一个同名的 Endpoints 对象，它保存了所有匹配标签选择器的 Pod 的 IP 地址和端口；
- 若单个 Endpoints 对象过大，则当其发生变化时需要处理大量的网络流量，从而影响集群中主控件的性能。因此，Kubernetes 引入
Endpoint Slice 资源解决这一问题。该资源建立在 Endpoint 的基础上，并为其提供了可伸缩扩展的替代方案。当集群中的 Service 关联有大量（>100）Endpoints 对象时，它们将被分成多个较小的 Endpoint Slice 对象。

### 无标签选择器的 Service

上文提到，Service 常用于对 Pod 的流量调度，但也可以用于访问：

- 同一集群不同 namespace 中的或其他集群中的 Service
- 其他的一些后端程序或数据库

此时需要定义一个没有标签选择器的 Service，如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

因为该 Service 没有标签选择器，相应的 Endpoint 对象便无法自动创建。因此我们需要手动创建一个 Endpoint 对象，以便将到达该 Service 监听的端口的请求映射到后端程序或其他 Service 的 IP 地址和端口：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

在上述例子中，Service 将请求路由到 Endpoint：192.0.2.42:9376 (TCP)。另外，目标 IP 不能是本机回环地址或虚拟 IP 地址。

### Headless Services

Headless Service 不提供负载均衡的特性，也没有自己的 IP 地址，kube-proxy 并不会处理这类 Service。只需要指定`spec.clusterIP`字段值为"None"，便可以创建一个 Headless Service 对象。

## 代理模式

- [**iptables**](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)：由 kube-proxy 负责为每一个 Service 创建和维护 iptables 的路由规则，其余工作由内核的 iptables 完成。当数据包发往 Service 时，iptables 承担了从 Service 的 IP 地址到 Pod 的 IP 地址的目的地址转换（DNAT）和负载均衡工作。在默认情况下，iptables 会将请求随机重定向到由 Service 代理的一组 Pod 中的某个 Pod 上。另外，iptables 还使用 Netfilter 的 conntrack 工具包记录选择的目标 Pod 的 IP 地址。这样当数据包返回时，iptables 便可以根据该记录，将返回数据包的源地址由 Pod 的 IP 地址转换为 Service 的 IP 的地址（SNAT）。

![demo.002](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/demo.002.png)

- **IPVS**：kube-proxy 监视 Service 和 Endpoint 对象的改变，调用 netlink 接口相应地创建 IPVS 规则，并定期地将 IPVS 规则与 Service 和 Endpoint 对象同步。IPVS 可以转发 TCP/UDP 请求到实际的服务器上，使得一组实际的服务器（Pod）看起来像是只通过一个单一 IP 地址（Service）访问的服务一样。

![234259911661134760](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/234259911661134760.jpg)

## 服务发现模式

Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS。

### 环境变量

kubelet 会为节点上活跃的 Service 对象创建一组环境变量（包括 Service 的 ClusterIP、监听的端口等），并在创建 Pod 时这些环境它们注入其中。使用这种服务发现模式要求 Service 先于 Pod 创建，因此有一定的局限性。

举例来说，一个名称为 redis-master 的 Service 暴露了 TCP 端口 6379，同时为它分配了 ClusterIP 地址 10.0.0.11。Pod 中的环境变量如下：

```json
REDIS_MASTER_SERVICE_HOST=10.0.0.11
REDIS_MASTER_SERVICE_PORT=6379
REDIS_MASTER_PORT=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
REDIS_MASTER_PORT_6379_TCP_PORT=6379
REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11
```

### DNS

集群中的 DNS 服务器（例如 CoreDNS）使用 Kubernetes 的 watch api 不断监测 Service 的创建并为每一个 Service 创建一条 DNS 记录，从而使 Pod 可以通过解析该记录而得到 Service 的 Cluster IP。对于不同类型的 Service，其 DNS 记录的分配方式有所不同：

#### Headless Service 以外的 Service

- Service 将被分配一个 A 记录，格式为：

  \<service-name\>.\<namespace-name\>.svc.cluster.local

  其中，cluster.local 为默认的集群域名，该 DNS 记录解析到 Service 的 ClusterIP。

- 若 Servic 存在一个已被命名的端口（`spec.ports.name`字段不为空），则该端口将被分配一个 SRV 记录，其格式为：
  \<\_port-name\>.\<\_port-protocol\>.\<service-name\>.\<namespace-name\>.svc.cluster.local

  则可根据该 SRV 记录发现该 Service 的命名端口及 IP 地址。

#### Headless Service

- 已定义标签选择器的 Headless Service：Endpoints Controller 在 Kubernetes API 中创建 Endpoints 对象，并修改 DNS 配置返回一个 A 记录（格式与普通的 Service 相同），指向该 Service 选取的一组 Pod 的 IP 地址。

- 无标签选择器的 Headless Service：Endpoints Controller 不再创建 Endpoints 对象。若 Service 类型为 ExternalName，DNS 服务返回其 CNAME 记录；若 Service 为其他类型，返回与该 Service 同名的 Endpoints 对象的 A 记录。Service 的类型将在下一节中进行介绍。

## 外部访问

集群中的节点（虚拟机）可以通过网关访问互联网，但 Pod 的 IP 地址与其所在节点的 IP 地址显然不同。由于网关的 NAT 功能只能转换节点（虚拟机）的 IP 地址，而无法转换 Pod 的 IP 地址。毕竟网关根本无法知晓节点上运行了什么样的 Pod，因此 Pod 是无法访问外网的。而我们常希望将集群中的一些运行前端应用的 Pod 暴露给集群外部的 IP 地址，这时便需要改变`spec.type`字段定义特殊类型的 Service 了。

### ClusterIP

默认的 Service 类型，通过集群中的内部 IP 暴露 Service，这种方式的 Service 只能在集群内部访问。

### NodePort

通过每个节点上的 IP 和静态端口（`spec.ports.nodePort`）暴露 Service，其配置如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: MyApp
  ports:
    - port: 80               
      targetPort: 80
      nodePort: 30007        # 如不指定，则会从 30000-32767 范围内分配一个端口号
```

通过、<NodeIP\>:`spec.ports.nodePort`向集群外部暴露 Service，同时也会在集群内部暴露 ClusterIP 类型的访问方式。当集群外部向节点的外网 IP 地址发送请求时，流量将会被路由到`spec.clusterIp:spec.ports.port`，最后再被转发到后端 Pod 的`spec.ports.targetPort`端口上。使用 NodePort 类型的 Service 来实现外部访问有以下缺陷：

- 当集群中有多个该类型的 Service 时，每个 Service 都要绑定一个节点端口。节点需要暴露外围端口调用 Service，可能造成管理混乱；
- 实现该类型的 Service 需为节点配置外网 IP，因此可能无法应用于很多公司要求的防火墙规则。

### LoadBalancer

使用云提供商的负载均衡器向集群外部暴露 Service，同时也会 [暴露 ClusterIP 和 NodePort 类型的访问方式](https://stackoverflow.com/questions/41509439/whats-the-difference-between-clusterip-nodeport-and-loadbalancer-service-types)：

- `spec.clusterIp`:`spec.ports.port`；
- \<NodeIP\>:`spec.ports.nodePort`；
- 负载均衡器的 IP 和 port。

负载均衡器是异步创建的。当 LoadBalancer 类型的 Service 创建完成后，负载均衡器的信息将被回写到 Service 的`status.loadBalancer`字段中，例如：

```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: example-service
  spec:
    selector:
      app: example
    clusterIP: 10.84.206.2
    externalIPs:              # 某些云服务商允许直接设置 loadBalancerIP 字段来指定 loadBalancer 的 IP 地址
    - 172.29.245.25             
    ports:
    - nodePort: 31688
      port: 8765
      protocol: TCP
      targetPort: 9376
    type: LoadBalancer
  status:
    loadBalancer:
      ingress:
      - ip: 172.29.245.25    
```

Loadbalancer 类型的 Service 处理外网入方向流量的流程如下：

- Loadbalancer 类型的 Service 创建后，Cloud Controller（云服务商提供）将为其创建一个负载均衡器；
- 负载均衡器只能直接和节点（虚拟机）沟通，并不知晓 Service 和 Pod 的存在。当数据包从请求方（互联网）到达负载均衡器之后，将被转发到集群中的某一节点的`spec.ports.nodePort`端口上；
- 与 NodePort 类型的 Service 相同，数据包再次被转发到`spec.clusterIp`:`spec.ports.port`，最后通过 iptables 发送到后端 Pod。

### ExternalName

ExternalName 类型的 Service 将 Service 映射到 DNS 名称，而非标签选择器选取的一组 Pod 上，其配置如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

当查找域名：my-service.prod.svc.cluster.local 时，集群内的 DNS 服务将返回一条 CNAME 记录，即`spec.externalName`字段的值 my.database.example.com。访问该 Service 的方式与其他类型的 Service 相同，但主要区别在于重定向发生在 DNS 级别，而不是通过代理或转发。

### External IP

如果有外部 IP 路由到 Kubernetes 集群的一个或多个节点，任意类型的 Service 可以通过、<`spec.externalIPs`\>:\<`spec.ports.port`\>进行访问。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
  externalIPs:
    - 80.11.12.10
```

在上面的例子中，客户端即可通过 80.11.12.10:80 访问名为 my-service 的 Service 对象。

## 参考文献

[A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)

[A Guide to the Kubernetes Networking Model](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)

[What's the difference between ClusterIP, NodePort and LoadBalancer service types in Kubernetes?](https://stackoverflow.com/questions/41509439/whats-the-difference-between-clusterip-nodeport-and-loadbalancer-service-types)
