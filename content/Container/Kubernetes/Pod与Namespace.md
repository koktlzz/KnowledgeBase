---
title: "Pod 与 Namespace"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  container:
    parent: "Kubernetes"
weight: 900
---

## Pod

### 基本特性

- Pod 是 Kubernetes 中的最小单位，由一个或一组容器组成。一个容器中运行一个进程，而一个 Pod 运行多个进程；
- 为实现亲密性应用，Pod 内部的容器共享网络和挂载卷，因此可以方便地实现多个应用间交互；
- 每一个 Pod 其实都具有两种不同的容器，一种是 Init 容器：在 Pod 启动时运行，主要用于初始化一些配置。另一种是 Pod 在 Running 状态时内部存活的应用容器，它们的主要作用是对外提供服务或者作为工作节点处理异步任务等；
- Pod 的生命周期短暂。

### Spec 和 Status

以一个名为 busybox 的 Pod 为例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  labels:
    app: busybox
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

Pod 的 Spec 指定了 Pod 中包含的容器以及容器的镜像、启动命令等信息，上述例子中还包括了镜像拉取策略、容器资源限制以及 Pod 重启策略。
而每一个 Pod 的 Status 包含了生命周期、当前服务状态、宿主机和 Pod 的 IP 地址以及其中内部所有容器的状态信息等：

```yaml
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-12-09T02:40:37Z
    status: "True"
    type: Initialized
  containerStatuses:
  - containerID: docker://99f668a89db97342d7bd603471dfad5be262d7708b48cb6c5c8e374e9a13cf4f
    image: busybox:latest
    imageID: docker-pullable://busybox@sha256:915f390a8912e16d4beb8689720a17348f3f6d1a7b659697df850ab625ea29d5
    lastState: {}
    name: busybox
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: 2018-12-09T02:40:37Z
  hostIP: 10.128.0.18
  phase: Running
  podIP: 10.4.0.28
  # ...
```

### 共享网络

![demo.001.jpeg.001.jpeg.001](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/demo.001.jpeg.001.jpeg.001.jpeg)

- Pod 在启动时会创建一个网络设备，即 Pause 容器，它会为 Pod 创建一个 [Network namespace](/docker/docker核心原理/namespace隔离/#network)；
- Pod 中的其他容器（如图中的 CTR1、CTR2）在启动时使用系统调用 [setns()](/docker/docker核心原理/namespace隔离/#setns) 加入 Pause 容器的 Network namespace 中，因此能够通过 localhost 互相访问彼此暴露的端口和服务；
- Kubenet 插件为每个节点创建 cbr0 网桥并为每一个 Pod 创建 veth 接口，从而实现 Pod Network namespace 与 Root Network namespace 之间的通信。

### 共享存储

- 一个 Pod 可以设置一组共享的存储卷（Volume），Pod 中的所有容器都可以访问该共享卷，从而允许这些容器共享数据；
- Volume 还允许 Pod 中的持久化数据保留下来，即使其中的容器需要重新启动；
- 在当前 Pod 出现故障或者滚动更新时，对应 Volume 中的数据并不会被清除，而是会在 Pod 重启后重新挂载到期望的文件目录中；
- 卷的挂载是 Pod 启动之前必须要完成的工作。

### 生命周期

![podlifecycle](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210130011905.png)

- Pod 的状态有：Pending、Running、Succeeded、Failed 和 Unkown，容器的状态有：Waiting、Running 和 Terminated；
- Pod 遵循一个预定义的生命周期，起始于 Pending 阶段，如果至少其中有一个应用容器正常启动，则进入 Running 阶段。如果 Pod 中的容器均成功正常退出，则 Pod 为 Succeeded 状态。若 Pod 中的容器均已退出但至少有一个容器因为发生错误而退出，则 Pod 为 Failed 状态；
- 任何给定的 Pod（由 UID 定义）从不会被“重新调度（rescheduled）”到不同的节点。相反，这一 Pod 可以被一个新的、几乎完全相同的 Pod 替换掉。如果需要，新 Pod 的名字可以不变，但是其 UID 会不同；
- 一个 Pod 的完整生命周期经历：Create -> Probe -> Running -> Shutdown -> Restart。

#### Create

Pod 启动前由 Init 容器来完成一些初始化操作。初始化完毕后，Init 容器运行至 Terminated，应用容器启动。在应用容器启动后可以执行一些特定的指令，称为启动后钩子 (PostStart)。Kubernetes 在管理容器时，将一直等到 PostStart 执行完成后，才会将容器的状态标记为 Running。

#### Probe

如果我们配置了合适的健康检查方法和规则，那么就不会出现服务未启动就被打入流量或者长时间未响应依然没有重启等问题。因此，我们应当尽可能为每一个 Pod 添加 livenessProbe（存活检查）和 readinessProbe（就绪检查）。ProbeManager 会根据 Pod 的配置分别选择调用 Exec、HTTPGet 或 TCPSocket 三种不同的 Probe 方式。

#### Shutdown

对于每一个容器来说，它们的停止有一下几个步骤：

1. 先从 Pod 的规格中计算出当前停止所需要的时间；
2. 然后调用 PreStop 的钩子方法，让容器中的应用程序能够有时间完成一些未处理的操作；
3. 随后调用远程的服务停止运行中的容器。

## Namespace

### 基本特性

- Namespace 隔离各种资源，可以理解为 kubernetes 内部的虚拟集群组；
- 不同 Namespace 内的资源 Name 可以相同，相同 Namespace 内的同种资源则不可以；
- Namespace 不能相互嵌套，每个 Kubernetes 资源只能在一个 Namespace 中；
- 默认的 Namespace：default, kube-system,kube-public。

### kubectl 命令

- 查看所有 namespace：

  ```bash
  kubectl get ns
  ```

- 查看指定 namespace 下的资源：

  ```bash
  kubectl get <resource> -n <namespace>
  ```

- 查看所有 namespace 下的资源：

  ```bash
  kubectl get <resource> --all-namespace=true
  ```

- 利用 Openshift 的 oc 命令切换 namespace：

  ```bash
  oc project <namespace>
  ```
