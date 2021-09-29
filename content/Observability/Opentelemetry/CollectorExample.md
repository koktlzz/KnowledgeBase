---
title: "Collector Example"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  observability: 
    parent: "Opentelemetry"
weight: 1600
---

## 环境准备

使用 [Sealos](https://github.com/fanux/sealos) 安装 Kubernetes 集群，至少需要三个 Master 节点 和 一个 Node 节点：

| Role    | Hostname       | IP            |
| ------- | -------------- | ------------- |
| master1 | koktlzz-k8s-01 | 10.122.70.135 |
| master2 | koktlzz-k8s-02 | 10.122.70.84  |
| master3 | koktlzz-k8s-03 | 10.122.70.113 |
| node1   | koktlzz-k8s-04 | 10.122.70.68  |

在安装前需要保证节点间时间同步，本例中使用 Chronyd 来实现。我们将 master1 作为 Chronyd Server，需要在`/etc/chrony.conf`文件中添加如下内容：

```shell
local stratum 10    ：即使 Server 无法从互联网同步时间，也同步本机时间至 Client
allow all           ：允许所有主机从 Server 同步时间
```

其他三台 Client 机器需要在`/etc/chrony.conf`文件中注释掉默认的 Server 配置，并指定 master1 作为自己的 Chronyd Server：

```shell
server 10.122.70.135 iburst
```

重启各台机器的 Chronyd 服务后，我们可以查看节点间的时间同步情况：

```shell
[root@koktlzz-k8s-01 ~]# chronyc activity
200 OK
4 sources online
0 sources offline
0 sources doing burst (return to online)
0 sources doing burst (return to offline)
0 sources with unknown address
[root@koktlzz-k8s-01 ~]# chronyc tracking
Reference ID    : 70414589 (112.65.69.137)
Stratum         : 3
Ref time (UTC)  : Fri Sep 17 01:53:34 2021
System time     : 0.000236189 seconds fast of NTP time
Last offset     : +0.000164347 seconds
RMS offset      : 0.000131223 seconds
Frequency       : 14.670 ppm slow
Residual freq   : +0.002 ppm
Skew            : 0.053 ppm
Root delay      : 0.008919094 seconds
Root dispersion : 0.028753694 seconds
Update interval : 1028.6 seconds
Leap status     : Normal
```

确认安装环境符合 [要求](https://github.com/fanux/sealos#%E8%A6%81%E6%B1%82%E5%92%8C%E5%BB%BA%E8%AE%AE) 后，我们便可以开始安装集群：

```shell
# 下载并安装 sealos, sealos 是个 golang 的二进制工具，直接下载拷贝到 bin 目录即可，release 页面也可下载
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin 

# 下载离线资源包
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/05a3db657821277f5f3b92d834bbaf98-v1.22.0/kube1.22.0.tar.gz

# 安装一个三 master 的 kubernetes 集群
$ sealos init --passwd '******' \
    --master 10.122.70.135  --master 10.122.70.84  --master 10.122.70.113  \
    --node 10.122.70.68 \
    --pkg-url /root/kube1.22.0.tar.gz \
    --version v1.22.0
```

Sealos 还支持部分组件的自定义安装、Pod CIDR 配置以及增删节点等操作，详见 [官方文档](https://www.sealyun.com/instructions)。

## 镜像仓库

在 Sealos 的离线安装包中已经包含了基础镜像，但缺少 Prometheus、Jaeger 和 Dashboard 等应用镜像。为了方便部署，我们在 node1 上搭建一个镜像仓库，此处使用 Docker Registry：

```shell
docker run -d -p 5000:5000 --name=registry --restart=always --privileged=true --log-driver=none -v /data/registry:/tmp/registry registry
```

镜像仓库地址为：[10.122.70.68:5000](http://10.122.70.68:5000/)。以上传 jaeger-operator 镜像为例，步骤如下：

```shell
# 导入镜像
docker load < jaeger-operator.tar
# 更改镜像 tag
docker tag docker.io/jaegertracing/jaeger-operator:1.25.0 10.122.70.68:5000/jaegertracing/jaeger-operator:1.25.0
# 配置 insecure 镜像仓库地址
cat << EOF > /etc/docker/daemon.json
> {"insecure-registries":["10.122.70.68:5000"]}
> EOF
# 上传镜像
docker push 10.122.70.68:5000/jaegertracing/jaeger-operator:1.25.0
```

浏览器访问  [10.122.70.68:5000/v2/catalog](http://10.122.70.68:5000/v2/_catalog) 即可看到上传成功的所有镜像。如果想要删除已上传的镜像，可以进入 Registry 容器进行操作：

```shell
docker exec registry  rm -rf /var/lib/registry/docker/registry/v2/repositories/<镜像名>
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

由于我们搭建的镜像仓库是非安全（insecure）的，因此集群中的应用想要拉取其中的镜像，还需要对默认的容器运行时 Containerd 进行相关配置，否则会触发报错：`http: server gave HTTP response to HTTPS client`。

首先生成 containerd 的默认配置文件：

```shell
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
```

然后修改`config.toml`文件中的 Registry 部分：

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.122.70.68:5000"]
      endpoint = ["http://10.122.70.68:5000"]
```

最后重启 containerd 服务，可使用`containerd config dump`来确认配置修改是否生效。

## Prometheus no otel target

```shell
level=error ts=2021-09-08T08:21:41.327Z caller=klog.go:116 component=k8s_client_runtime func=ErrorDepth msg="pkg/mod/k8s.io/client-go@v0.21.3/tools/cache/reflector.go:167: Failed to watch *v1.Pod: failed to list *v1.Pod: pods is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list resource \"pods\" in API group \"\" in the namespace \"observability\""
```

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prometheus-k8s
  namespace: monitoring
rules:
- apiGroups: [""]
  resources: ["services","pods","endpoints","nodes/metrics"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "watch", "list"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get", "watch", "list"]
```

## 参考文献

[fanux/sealos](https://github.com/fanux/sealos)

[insecure local registry gives "http: server gave HTTP response to HTTPS client"](https://github.com/containerd/cri/issues/1367)

[Containerd 使用教程 – 云原生实验室](https://fuckcloudnative.io/posts/getting-started-with-containerd/)

[Opentelemetry Collector的配置和使用](https://www.cnblogs.com/charlieroro/p/13883602.html)

[opentelemetry-go/example/otel-collector](https://github.com/open-telemetry/opentelemetry-go/tree/main/example/otel-collector)
