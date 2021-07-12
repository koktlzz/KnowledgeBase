---
title: "Network Policy"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  kubernetes:
    parent: "Kubernetes 进阶"
weight: 600
---

## 概述

如果希望在 IP 地址或端口层面（OSI 第 3 层或第 4 层）控制网络流量，那么可以为集群中的特定应用使用 Kubernetes 网络策略（NetworkPolicy）。能够与 Pod 通信的对象是通过以下三个标识符的组合来规定的：

- 其他被允许的 Pods（Pod 无法阻塞对自身的访问）
- 被允许的 Namespace
- IP/port（与 Pod 运行所在的节点的通信总是被允许的，无论是 Pod 还是节点的 IP 地址）

当同一个 Pod 被多个 Network Policy 通过标签选择器选择时，其网络策略为多个 Network Policy 的并集。

## 配置

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default # namespace
spec:
  podSelector:    # 若为空，选择 default-namespace 下的所有 pod
    matchLabels:
      role: db
  policyTypes:
  - Ingress      # 入站策略
  - Egress       # 出站策略
  ingress:
  - from:
    - ipBlock:   # 白名单 ip 网段
        cidr: 172.17.0.0/16
        except:  # 黑名单 ip 网段
        - 172.17.1.0/24
    - namespaceSelector:  # 带有 "project=myproject" 标签的所有 namespace 中的 Pod
        matchLabels:
          project: myproject
    - podSelector:        # "default" namespace 下带有 "role=frontend" 标签的所有 Pod
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:          # 允许目标 ip 为 CIDR 10.0.0.0/24 且目标端口为 5978 TCP 端口的连接。
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

### 拒绝所有入方向

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### 允许所有入方向

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
  policyTypes:
  - Ingress
```

### 允许所有出方向

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

### 禁止所有流量出入

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```
