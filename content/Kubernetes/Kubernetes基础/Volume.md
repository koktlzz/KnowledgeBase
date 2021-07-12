---
title: "Volume"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  kubernetes:
    parent: "Kubernetes 基础"
weight: 500
---

## 意义

Kubernetes 引入 Volume 资源来解决以下问题：

- 容器中的文件在磁盘上是临时存放的，kubelet 重启容器后，文件将会丢失；
- 在运行多个容器的 Pod 内实现文件共享。

## 配置

一个典型的有挂载卷的 Pod 配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: docker.io/nginx:latest
    name: nginx
    volumeMounts:
    - mountPath: /nginx-master-data
      name: test-volume
  volumes:
  - name: test-volume
    emptyDir: {}
```

- `spec.containers.volumeMounts.mountPath`字段定义了 Volume 在容器中的挂载位置；
- 字段`spec.containers.volumeMounts`与`spec.volumes.name`必须相同；
- `spec.volumes.emptyDir`代表 Volume 的类型。

## 类型

Volume 的核心是包含一些数据的一个目录，Pod 中的容器可以访问该目录。Volume 的类型将决定该目录如何形成的、使用何种介质保存数据以及目录中存放的内容。

![20210313162624](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210313162624.png)

### emptyDir

在当前 Pod 对应的目录创建了一个空的文件夹，这个文件夹会随着 Pod 的删除而删除。

### hostPath

将 Pod 所在节点文件系统上的文件或目录挂载到 Pod 中。

### nfs

将 NFS（Network File System）挂载到 Pod 中，可以在不同节点上的不同 Pod 之间共享数据并被多个 Pod 同时读写。当 Pod 被移除时，将仅仅卸载 nfs 数据卷，Volume 中的数据仍将被保留；

### configMap

提供了向 Pod 注入配置数据的方法。ConfigMap 对象中存储的数据可以被 configMap 类型的卷引用，然后被 Pod 中运行的应用使用。其典型配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_key
            path: log_path
```

名为 log-config 的 ConfigMap 对象以卷的形式挂载到 Pod 中。其中，挂载的目录为 ConfigMap 对象中键为 log_key 的所有内容（`spec.volumes.configMap.items.key`），路径为 Pod 中的/etc/config/log_path（`spec.containers.volumeMounts.mountPath`/`spec.volumes.configMap.items.path`）。

### secret

secret 类型的数据卷可以用来注入敏感信息（例如密码）到 Pod 中。

### persistentVolumeClaim（PVC）

常见的 Volume 类型，如 emptyDir、hostPath、configMap 和 secret 等，与所属的 Pod 具有相同的生命周期。为了实现数据的持久化存储，我们需要使用 persistentVolumeClaim 类型的 Volume“申领”持久卷 PV（Persistent Volume）并挂载到 Pod 中。

PV、PVC 和存储类（Storage Class）三者之间的关系如下：

- PV 是集群中的持久化存储资源，通常由集群管理员创建和管理；

- Storage Class 用于用于描述集群中可以提供的 PV 的类型。若 PVC 关联了 Storage Class 且配置符合要求，Storage Class 也可以根据自身配置中的`provisioner`、`parameters`和`reclaimPolicy`字段动态创建 PV；

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: standard
  provisioner: kubernetes.io/aws-ebs
  parameters:
    type: gp2
    fsType: ext4
  reclaimPolicy: Retain
  allowVolumeExpansion: true
  mountOptions:
    - debug
  volumeBindingMode: Immediate
  ```

- Storage Class 配置中的`provisioner`字段定义了使用哪种数据卷插件（如 AWS EBS）来创建 PV（大多数由云服务商提供），`parameters`字段则定义了数据卷类型（如 gp2）、文件系统类型（如 ext4）等；

- Storage Class 配置中的`reclaimPolicy`字段定义了 PV 的回收策略，默认为 Delete，代表移除 PV 及其关联的外部存储介质。若设置为 Retain，则 PVC 删除后 PV 及 PV 中的数据依然存在，只是不会再被其他 PVC 申领；

- PVC 是申领 PV 资源的请求，通常由应用程序发出并指定 Storage Class 和需求的 Volume 空间大小；

![193990293059406992](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/193990293059406992.jpg)

### Volume Snapshot

- Volume Snapshot 是用户对存储卷创建快照的请求，相当于 PVC；
- Volume Snapshot Content 是根据集群中已存在的存储卷（如 PVC）创建的快照资源，相当于 PV。因此，PVC 也可以反过来根据 Volume Snapshot Content 初始化；
- Volume Snapshot Class 指定了 Volume Snapshot 的属性，相当于 Storage Class。即使从相同存储卷获取的快照，也可能因 Volume Snapshot Class 的配置不同而有所不同。

一个简单的 Volume Snapshot 配置如下，该对象将根据指定的 PVC 动态创建 Volume Snapshot Content：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-test
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass   # 指定 Volume Snapshot Class
  source:
    persistentVolumeClaimName: pvc-test             # 指定快照的数据源
```

当数据源不为 PVC 时，Volume Snapshot 也可以根据预定义的方式创建 Volume Snapshot Content:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  source:
    volumeSnapshotContentName: test-content    
```

该方法需提前定义好对应的 Volume Snapshot Content 配置：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: test-content
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002    
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

`spec.source.snapshotHandle`字段指定了 Volume Snapshot Content 的数据源，其值为存储后端创建卷时返回的唯一标识符。

## 数据卷内子路径

定义`spec.containers.volumeMounts.subPath`字段，可以使容器在挂载 Volume 时指向数据卷内部的一个子路径，而不是直接指向数据卷的根路径。在下面的例子中，一个运行 LAMP（Linux Apache Mysql PHP）应用的 Pod 使用了一个共享数据卷，HTML 内容映射到数据卷内部的 html 目录，而数据库的内容映射到了 mysql 目录：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
        readOnly: false
    - name: php
      image: php:7.0-apache
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
        readOnly: false
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

## 挂载传播

Volume 的传播能力允许将容器挂载的 Volume 共享到同一 Pod 中的其他容器，甚至共享到同一节点上的其他 Pod。由 Pod 配置中的`spec.containers.volumeMounts.mountPropagation`字段控制。可选的取值有：

- None：默认值。在数据卷被挂载到容器之后，此挂载卷将不会接受主机或其他容器后续在此卷或其任何子目录上执行的挂载操作。类似的，容器所创建的卷挂载在主机上也不可见。
- HostToContainer：在数据卷被挂载到容器之后，宿主机向该数据卷对应目录添加挂载时，容器是可见的。
- Bidirectional：在数据卷被挂载到容器之后，宿主机向该数据卷对应目录添加挂载时，容器是可见的；同时，从容器中向该数据卷创建挂载，宿主机同样可见。如果在容器内进行不合适的挂载，可能影响宿主机的操作系统正常执行。
