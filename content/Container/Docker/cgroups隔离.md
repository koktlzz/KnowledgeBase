---
title: "CGroups 隔离"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  container:
    parent: "Docker"
weight: 500
---

## 作用

cgroups(controll groups) 可以把一系列系统任务及其子任务（线程/进程）划分为等级不同的组，从而对任务组使用的物理资源（CPU、内存、IO 等）进行限制和记录。

- 资源限制：对任务使用的资源总额进行限制，一旦超过这个配额就发出相关提示；
- 优先级分配：通过分配的 CPU 时间片数量及磁盘 IO 带宽大小，实际上就相当于控制了任务运行的优先级；
- 资源统计：cgroups 可以统计系统的资源使用量，如 CPU 使用时长，内存用量等；
- 任务控制：cgroups 可以对任务执行挂起、恢复等操作。

## 组织结构

cgroups 的组织结构中有以下几种元素：

- 任务（task）：系统的一个进程或线程；

- 控制组（cgroup）：按某种资源控制标准划分而成的任务组，一个任务可以加入某个 cgroup，也可以在不同 cgroup 中迁移；

- 子系统（subsystem）：即资源调度器，如 CPU 子系统可以控制 CPU 时间分配；

- 层级（hierarchy）：由一系列控制组以一个树状结构排列而成，每个层级通过绑定对应的子系统进行资源控制。

它们之间的关系需要遵循一定的规则：

- 同一个层级可以绑定多个子系统；
- 一个已经有层级绑定的子系统，不能附加到其他含有别的子系统的层级之上；
- 一个任务不能存在于同一层级的不同控制组中；
- 调用 fork() 或 clone() 方法后，父子任务（进程）的控制组互不影响。

![20210201012614](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210201012614.png)

在上图的 cgroups 结构中，左侧的层级同时绑定了 CPU 子系统和 Memory 子系统；由于此时 Memory 子系统已经有层级绑定，因此无法附加到一个已经绑定了 Net_cls 子系统的层级上；如果任务 task1 处于左侧层级下的 cg1 控制组中，那么便无法加入同一层级下的 cg2 控制组，但是可以加入右侧层级的 cg3 控制中。

## 实现

cgroup 的实现形式表现为一个文件系统，可以通过 **mount -t cgroup** 命令查看各种子系统的挂载情况：

```bash
[root@koktlzz ~]# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
```

以 CPU 子系统为例，若在其挂载的目录下创建一个目录 newcg，那么便创建了一个新的控制组且系统会自动在其中生成一些的控制组文件：

```bash
[root@koktlzz ~]# cd /sys/fs/cgroup/cpu
[root@koktlzz cpu]# mkdir newcg
[root@koktlzz cpu]# ls newcg
cgroup.clone_children  cpuacct.usage_all          cpuacct.usage_sys   cpu.rt_period_us   notify_on_release
cgroup.procs           cpuacct.usage_percpu       cpuacct.usage_user  cpu.rt_runtime_us  tasks
cpuacct.stat           cpuacct.usage_percpu_sys   cpu.cfs_period_us   cpu.shares
cpuacct.usage          cpuacct.usage_percpu_user  cpu.cfs_quota_us    cpu.stat
```

而 CPU 子系统的挂载目录中，本身就包含了这类文件。同时还有一些控制组目录，如 docker：

```bash
[root@koktlzz cpu]# ls /sys/fs/cgroup/cpu
assist                 cpuacct.usage              cpuacct.usage_sys   cpu.rt_runtime_us  newcg              user.slice
cgroup.clone_children  cpuacct.usage_all          cpuacct.usage_user  cpu.shares         notify_on_release
cgroup.procs           cpuacct.usage_percpu       cpu.cfs_period_us   cpu.stat           release_agent
cgroup.sane_behavior   cpuacct.usage_percpu_sys   cpu.cfs_quota_us    docker             system.slice
cpuacct.stat           cpuacct.usage_percpu_user  cpu.rt_period_us    init.scope         tasks
```

Docker daemon 首先会在子系统的挂载目录下创建一个名为 docker 的控制组，然后在 docker 控制组中再为每个容器创建一个以容器 id 命名的子控制组。即使上层控制组的文件系统被卸载，下层的子控制组配置依然有效。docker 控制组的层级结构如下所示：

```bash
[root@koktlzz docker]# tree /sys/fs/cgroup/cpu/docker
/sys/fs/cgroup/cpu/docker
├── cgroup.clone_children
├── cgroup.procs
├── cpuacct.stat
├── cpuacct.usage
├── cpuacct.usage_all
├── cpuacct.usage_percpu
├── cpuacct.usage_percpu_sys
├── cpuacct.usage_percpu_user
├── cpuacct.usage_sys
├── cpuacct.usage_user
├── cpu.cfs_period_us
├── cpu.cfs_quota_us
├── cpu.rt_period_us
├── cpu.rt_runtime_us
├── cpu.shares
├── cpu.stat
├── d682e9c84e04ebe41461fdf3aa6d22903c68cf2260218d1051af13e2bb1a0edd
│   ├── cgroup.clone_children
│   ├── cgroup.procs
│   ├── cpuacct.stat
│   ├── cpuacct.usage
│   ├── cpuacct.usage_all
│   ├── cpuacct.usage_percpu
│   ├── cpuacct.usage_percpu_sys
│   ├── cpuacct.usage_percpu_user
│   ├── cpuacct.usage_sys
│   ├── cpuacct.usage_user
│   ├── cpu.cfs_period_us
│   ├── cpu.cfs_quota_us
│   ├── cpu.rt_period_us
│   ├── cpu.rt_runtime_us
│   ├── cpu.shares
│   ├── cpu.stat
│   ├── notify_on_release
│   └── tasks
├── notify_on_release
└── tasks
```

其中的 d682e9c84e04ebe41461fdf3aa6d22903c68cf2260218d1051af13e2bb1a0edd 控制组是一个正在运行的 nginx 容器 id：

```bash
[root@koktlzz docker]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED      STATUS      PORTS                NAMES
d682e9c84e04   nginx     "/docker-entrypoint.…"   2 days ago   Up 2 days   0.0.0.0:80->80/tcp   nginx
```

这个容器内的进程 pid 都会写入到该控制组的 tasks 目录中，把任务的 TID（即线程或进程的 id）写入到这个文件就意味着将这个任务加入到该控制组中：

```bash
[root@koktlzz docker]# cat d682e9c84e04ebe41461fdf3aa6d22903c68cf2260218d1051af13e2bb1a0edd/tasks
2452208
2452318
[root@koktlzz docker]# ps -ef | grep 2452208
UID          PID    PPID  C STIME TTY          TIME CMD
root     2452208 2452188  0 1 月 29 ?       00:00:00 nginx: master process nginx -g daemon off;
101      2452318 2452208  0 1 月 29 ?       00:00:00 nginx: worker process
root     2689799 2687314  0 02:29 pts/0    00:00:00 grep --color=auto 2452208
```

Docker daemon 通过在控制文件（如 cpu.cfs_quata_us）中写入预设的资源配额实现 cgroups 的限制功能，其中 5000 代表限制 CPU 的最高使用率为 50%：

```bash
[root@koktlzz docker]# docker run -d --cpu-quota=50000 nginx
85ff8f8f779067c4938a011420d9afdb76baa3f69b17d76c0cfe9b8f716b228f
[root@koktlzz docker]# cat 85ff8f8f779067c4938a011420d9afdb76baa3f69b17d76c0cfe9b8f716b228f/cpu.cfs_quota_us
50000
```
