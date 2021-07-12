---
title: "Workloads"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  kubernetes:
    parent: "Kubernetes 基础"
weight: 600
---

## 简介

Workloads（工作负载）资源是 Kubernetes 上运行的应用程序，可以用来创建和管理多个 Pod，提供副本管理、健康检查、滚动升级和集群级别的自愈能力。例如，如果一个节点故障，Workloads 配置的 Controller 就能自动将该节点上的 Pod 调度到其他健康的节点上。Workloads Controller 运行在 Kubernetes 集群的主节点上，它们不断控制集群中的资源向期望状态迁移（stauts -> spec）。常用的 Workloads 资源有：

![20210313161059](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210313161059.png)

## ReplicaSet

决定一个 Pod 有多少同时运行的副本，并保证这些副本的期望状态与当前状态一致。

### 配置

一个典型的 ReplicaSet 配置如下：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3            # 副本数
  selector: 
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: docker.io/redis:latest
```

- 在上述配置信息中，字段`spec.template.metadata.labels`的值必须与`spec.selector`值相匹配，否则创建请求会被 Kubernetes API 拒绝；
- 被 ReplicaSet 控制的 Pod 在创建或更新后，其`metadata.ownerReferences`字段会添加该 ReplicaSet 的信息；
- 一旦删除了原来的 ReplicaSet，就可以创建一个新的来替换它。只要新旧 ReplicaSet 的`spec.selector`字段是相同的，新的 ReplicaSet 便会接管原有的 Pod。然而，修改 ReplicaSet 中的 template 并不会使其接管的 Pod 的 Spec 更新。

### 应用场景

- 重调度：保证指定数量的 Pod 正常运行；
- 弹性伸缩：修改`spec.replicas`字段，实现 Pod 数量弹性伸缩；
- 应用多版本追踪：修改`spec.selector`字段，实现对一个 Pod 的多版本管理。

由于 ReplicaSet 并不支持使用命令 **kubectl roll-update** 对 Pod 进行滚动更新，因此若想要以可控的方式更新 Pod，建议使用 Deployment。

## Deployment

Deployment 为 Pod 和 ReplicaSet 提供声明式的更新能力，用于管理无状态的应用。

### 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
```

- `.spec.selector`字段必须匹配`.spec.template.metadata.labels`，否则请求会被 Kubernetes API 拒绝；
- 当 Pod 的标签和 Deployment 的标签选择器匹配，但其模板和`.spec.template`不同，或者此类 Pod 的总数超过`.spec.replicas`的设置时，Deployment 会将其终止；
- 如果 Pod 的总数未达到期望值，Deployment 会基于`.spec.template`创建新的 Pod。
  
### 创建

```bash
[root@test-master1 ~]# kubectl create -f test.yml --record
deployment.apps/nginx-deployment created
[root@test-master1 ~]# kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           9s
```

查看上线状态

```bash
[root@test-master1 ~]# kubectl rollout status deployment.v1.apps/nginx-deployment
deployment "nginx-deployment" successfully rolled out
[root@test-master1 ~]# kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-5cbbd6c556-gj4vp   1/1       Running   0          4m        app=nginx,pod-template-hash=1766827112
nginx-deployment-5cbbd6c556-vk4vl   1/1       Running   0          4m        app=nginx,pod-template-hash=1766827112
nginx-deployment-5cbbd6c556-z24z8   1/1       Running   0          4m        app=nginx,pod-template-hash=1766827112
[root@test-master1 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-5cbbd6c556   3         3         3         2d
```

查看详细信息

```bash
[root@test-master1 ~]# kubectl describe rs nginx-deployment-5cbbd6c556
Name:           nginx-deployment-5cbbd6c556
Namespace:      gwr
Selector:       app=nginx,pod-template-hash=1766827112
Labels:         app=nginx
                pod-template-hash=1766827112
...
```

其中一些参数的含义：

- NAME：列出了集群中 Deployment 的名称；
- READY：应用程序的可用的副本数，显示的格式是“就绪个数/期望个数”；
- UP-TO-DATE：为了达到期望状态已经更新的副本数；
- AVAILABLE：显示应用可供用户使用的副本数；
- AGE：显示应用程序运行的时间。

我们可以发现 ReplicaSet 被命名为 Deployment 名称加一个数字的格式（nginx-deployment-5cbbd6c556），这个数字是使用 Pod 标签中`pod-template-hash`字段作为种子随机生成的。而此标签字段是通过对 Pod 的 template 进行哈希处理得到的，可确保 Deployment 管理的 ReplicaSet 不重叠。

### 更新

使用 **kubectl edit deployment \<deployment-name\>** 命令，可直接对 Deployment 管理的 Pod 进行更新。当使用 **kubectl describe deployments** 命令查看更新信息时，可以在 Events 下看到更新的过程：

```bash
Events:
    Type    Reason             Age   From                   Message
    ----    ------             ----  ----                   -------
    Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-2035384211 to 3
    Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 1
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 2
    Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 2
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 1
    Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1564180365 to 3
    Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-2035384211 to 0
```

- 当第一次创建 Deployment 时，它自动创建了一个 ReplicaSet（nginx-deployment-2035384211）并将其管理的 Pod 扩容至 3 个副本；
- 更新 Deployment 时，它又创建了一个新的 ReplicaSet（nginx-deployment-1564180365），并将其管理的 Pod 数量设置为 1，然后将旧 ReplicaSet 管理的 Pod 缩容到 2，以便至少有 2 个 Pod 可用且最多创建 4 个 Pod；
- 然后，它使用相同的滚动更新策略继续对新的 ReplicaSet 扩容并对旧的 ReplicaSet 缩容；
- 最终，新 ReplicaSet 管理的 Pod 副本数扩容至 3 个，旧 ReplicaSet 管理的 Pod 全部终止，更新完成；
- 在整个更新过程中，最多只有一个 Pod 副本不提供服务，且同一时刻不会有过多的 Pod 副本同时运行（默认最多比预期值多一个）。

### 回滚

当 Deployment 不稳定时（例如进入反复崩溃状态），我们需要对其进行回滚操作。默认情况下，Deployment 的所有上线记录都保留在系统中，以便可以随时回滚。

- 查看历史版本：**kubectl rollout history deployment \<deployment-name\>**
- 回滚至历史版本：**kubectl rollout undo deployment \<deployment-name\> --to-revision=\<deployment-version\>**

## StatefulSet

StatefulSet 用于管理有状态的应用程序，被其管理的 Pod 有一个按顺序增长的 ID。它与 Deployment 最大的不同在于，StatefulSet 始终将一系列不变的名字分配给 Pod，这些 Pod 从同一个模板创建但并不能相互替换且每个 Pod 都对应一个特有的持久化存储标识。

### 应用场景

- 每个 Pod 拥有稳定的、唯一的网络标识符（DNS Name）
- 每个 Pod 拥有稳定的、持久的存储（PersistentVolume）
- 有序的、优雅的部署和缩放
- 有序的、自动的滚动更新

### 配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10 # not 0
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

- 名为 nginx 的 Headless Service 用来给每个 Pod 配置 DNS Name；
- 名为 web 的 StatefulSet 中的字段`spec.replicas`和`spec.template.spec`字段表明将在独立的 3 个 Pod 副本中启动 nginx 容器；
- `volumeClaimTemplates`字段表明将通过 PersistentVolumes（PV）来为 Pod 提供持久化存储。在这种配置下，每个 StatefulSet 管理的 Pod 将分别挂载不同的 PV。而对于 Deployment 中的 Pod 来说，它们则将共享同一个 PV。

### Pod 标识

StatefulSet 管理的 Pod 具备一个唯一标识，该标识由以下几部分组成：

- 序号：假设一个 StatefulSet 的副本数为 N，其中的每一个 Pod 都会被分配一个序号，序号的取值范围从 0 到 N-1，并且该序号在 StatefulSet 内部是唯一的。
- 稳定的网络标识：
  - StatefulSet 中每个 Pod 根据 StatefulSet 的名称和 Pod 的序号派生出它的主机名 hostname：\<StatefulSet name\>\-\<Pod 序号、>；
  - StatefulSet 可以使用 Headless Service 来控制其 Pod 所在的域，该域（domain）的格式为：\<Service name\>.\<namespace\>.svc.cluster.local（"cluster.local"是集群的域名）；
  - StatefulSet 中每一个 Pod 将被分配一个 DNS Name，格式为：\<Pod name\>.\<所在域名、>，因此可以直接通过该 Pod 的 DNS Name 访问到 Pod。
- 稳定的存储：Kubernetes 为每个 VolumeClaimTemplate 创建一个 PersistentVolume。

### 部署和扩缩容

在默认情况下`.spec.podManagementPolicy`字段值为 OrderedReady，它代表依次进行 Pod 的部署和扩缩容：

- 在创建一个副本数为 N 的 StatefulSet 时，其 Pod 将被按{0...N-1}的顺序逐个创建；
- 在删除一个副本数为 N 的 StatefulSet（或其中所有的 Pod）时，其 Pod 将按照相反的顺序（即 {N-1...0}）终止和删除；
- 在对 StatefulSet 执行扩容操作时，新增 Pod 所有的前序 Pod 必须处于 Running（运行）和 Ready（就绪）的状态；
- 终止和删除 StatefulSet 中的某一个 Pod 时，该 Pod 所有的后序 Pod 必须全部已终止。

若要并行管理 Pod，需要设置`.spec.podManagementPolicy`字段值为 Parallel，此时 StatefulSet 将同时并行地创建或终止其所有的 Pod。

### 更新

StatefulSet 的更新策略有两种，它是通过定义`spec.updateStrategy.type`字段的方式进行选择的。

- On Delete：Controller 将不会自动更新 StatefulSet 中的 Pod, 用户必须手动删除 Pod 以便让 StatefulSet 创建新的 Pod，以此来对`spec.template`的变动作出反应。
- Rolling Updates
  StatefulSet 会删除和重建 StatefulSet 中的每个 Pod, 它将按照与 Pod 终止相同的顺序（从最大序号到最小序号）进行，每次更新一个 Pod。它会等到被更新的 Pod 进入 Running 和 Ready 状态，然后再更新其前序 Pod。

若 Pod 的 template 出错，导致 Pod 始终不能进入 Running 和 Ready 的状态，StatefulSet 将停止滚动更新并一直等待（OrderedReady）。在修复 template 以后，StatefulSet 将继续等待出错的 Pod 进入就绪状态，而该状态将永远无法出现。因此还必须删除所有已经尝试使用错误 template 的 Pod，随后 StatefulSet 才会使用修复后的 template 重建 Pod。

## DaemonSet

DaemonSet 确保全部（或者某些）节点上运行一个 Pod 的副本，且当有节点加入集群时，也会为他们新增一个 Pod。当有节点从集群移除时，这些 Pod 同样会被回收。删除 DaemonSet 将会删除它所创建的所有 Pod。

### 使用场景

- 在每个节点上运行集群守护进程 glusterd、ceph 等
- 在每个节点上运行日志收集守护进程 fluentd、logstash 等
- 在每个节点上运行监控守护进程 Prometheus 等

### Spec 配置

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

- `spec.template.spec.nodeSelector`字段指定运行 Pod 的节点，若不设置则默认在全部节点上运行；
- `spec.template.spec.restartPolicy`字段的值默认是 always，也必须是 always。

### 调度策略

DaemonSet Controller 将会向 DaemonSet 管理的的 Pod 添加`spec.nodeAffinity`字段，并进一步由 Kubernetes Scheduler 将 Pod 绑定到目标节点。

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - target-host-name
```

此外，容忍度（toleration）`node.kubernetes.io/unschedulable:NoSchedule`将被系统自动添加到 DaemonSet 的 Pod 中。由此，默认调度器在调度 DaemonSet 的 Pod 时可以忽略节点的 unschedulable 属性。

### 通信

与 DaemonSet 中的 Pod 进行通信的几种可能模式如下：

- Push：配置 DaemonSet 中的 Pod，将更新发送到另一个服务，例如统计数据库；

- 节点 IP 和已知端口：DaemonSet 中的 Pod 可以使用节点的端口，从而可以通过节点 IP 访问到 Pod。客户端能通过某种方法获取节点 IP 列表，并且基于此也可以获取到相应的端口；

- DNS：创建 Headless Service 并通过设置标签选择器选取 Pod，通过使用 Endpoints 对象或从 DNS 服务中检索到多个 A 记录来发现 DaemonSet；

- Service：创建 Service 并通过设置标签选择器选取 Pod，使用该 Service 随机访问到某个节点上的 DaemonSet。由于 Service 的负载均衡机制，因此没有办法访问到特定节点。

## Jobs

Job 会创建一个或者多个 Pods，并确保指定数量的 Pod 可以执行到 Succeeded 状态。随着 Pods 成功结束，Job 跟踪记录成功完成的 Pods 个数。当数量达到指定的成功个数阈值时，Job 结束。删除 Job 的操作会清除其创建的全部 Pod。

### 配置

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4 # Job 最大的重试次数
```

- `spec.template.spec.restartPolicy`字段定义了 Pod 的重启策略，此处只允许使用 Never 和 OnFailure 两个取值；
- `spec.completions`和`spec.parallelism`字段值默认为 1，代表 Job 只启动一个 Pod;
- `spec.completions`表示 Job 期望成功运行至结束的 Pod 数量。当该字段值大于 1 时，Job 将创建至相应数量的 Pod，编号为 1-`spec.completions`；
- `spec.parallelism`表示并行运行的 Pod 数量。当该字段值大于 1 时，Job 会依次创建相应数量的 Pod 并行运行，直到`spec.completions`个 Pod 成功运行至结束；
- `spec.activeDeadlineSeconds`字段指定 Job 的存活时长，该字段的优先级高于`spec.backoffLimit`；
- Job 终止后 Pod 不会被删除，可以通过定义`spec.ttlSecondsAfterFinished` 字段实现自动清理 Job 以及 Job 管理的 Pod。若字段值为 100，则代表 100 秒后清理。若字段值为 0，则代表 Job 完成后立即清理。

## Garbage Collection(GC)

在 Kubernetes 中，每一个从属对象都有一个`metadata.ownerReferences`字段，标识其拥有者是哪一个对象。GC Controller 会删除那些曾经有 owner，后来又不再有 owner 的对象。
