---
title: "Namespace 隔离"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  docker:
    parent: "Docker 核心原理"
weight: 700
---

## 简介

在 Linux 中可以使用 **lsns** 命令查看系统中的所有 namespace：

```bash
[root@koktlzz ~]# lsns
        NS TYPE   NPROCS   PID USER   COMMAND
4026531835 cgroup     94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531836 pid        94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531837 user       94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531838 uts        94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531839 ipc        94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531840 mnt        88     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026531860 mnt         1    16 root   kdevtmpfs
4026531992 net        94     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 17
4026532193 mnt         1   599 root   /usr/lib/systemd/systemd-udevd
4026532195 mnt         2   693 root   /sbin/auditd
4026532196 mnt         1   782 chrony /usr/sbin/chronyd
4026532197 mnt         1   893 root   /usr/sbin/NetworkManager --no-daemon
```

| Namespace | 系统调用参数  | 隔离内容                   | 应用意义                                                     |
| --------- | ------------- | -------------------------- | ------------------------------------------------------------ |
| UTS       | CLONE_NEWUTS  | 主机与域名                 | 每个容器在网络中可以被视作一个独立的节点，而非宿主机的一个进程 |
| IPC       | CLONE_NEWIPC  | 信号量、消息队列和共享内存 | 隔离容器间、容器与宿主机之间的进程间通信                     |
| PID       | CLONE_NEWPID  | 进程编号                   | 隔离容器间、容器与宿主机之间的进程 PID                        |
| Network   | CLONE_NEWNET  | 网络设备、网络栈、端口等   | 避免产生容器间、容器与宿主机之间产生端口已占用的问题         |
| Mount     | CLONE_NEWNS   | 挂载点（文件系统）         | 容器间、容器与宿主机之间的文件系统互不影响                   |
| User      | CLONE_NEWUSER | 用户和用户组               | 普通用户（组）在容器内部也可以成为超级用户（组），从而进行权限管理 |

## 进行 Namespace API 操作的方式

### clone()

clone() 可以在创建新进程（子进程）的同时创建 namespace，是 Docker 使用 namespace 最基本的方法。

```c
// child_func: 子进程运行的程序主函数
// child_stack: 子进程所使用的堆栈的位置，通常指向为子堆栈设置的内存空间的最高地址
// flags: 使用的 CLONE_*标志位
// args: 传入子进程的参数
// 在父进程中返回创建的子进程 pid
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

相比于调用 fork() 创建新进程，clone() 更加灵活，因为它可以通过改变 flags 参数来控制其实现的功能，后续将详细介绍各种 flags 的用法。

### setns()

setns() 可以加入一个已经存在的 namespace，Docker 中的 exec 命令便调用了该方法：

```c
// fd: 加入的 namespace 的文件描述符
// nstype: 检查 fd 指向的 namespace 类型是否符合实际要求，为 0 表示不检查
int setns(int fd, int nstype);
```

参数 fd 表示要加入的 namespace 的文件描述符，它指向 **/proc/[pid]/ns** 文件夹下中的软链接文件。而这些软连接文件又指向不同的 namespace, 如'cgroup:[4026531835]'中的数字即为其指向的 namespace 号 ($$是 bash 中表示当前运行的进程 PID)。

```bash
[root@koktlzz proc]# ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 root root 0 1 月  29 10:57 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 1 月  29 10:57 uts -> 'uts:[4026531838]'
```

参数 fd 是通过打开这些软链接文件得到的，其中参数 O_RDONLY 代表以只读的方式打开文件。

```c
fd = open(link_file, O_RDONLY);
```

即使一个 namespace 下的所有进程全部结束，我们也可以通过指向该 namespace 的文件描述符 fd 定位并加入其中，这也是 Docker 实现加入已存在 namespace 的最基本方式。

### unshare()

相比于 clone()，unshare() 不会启动一个新进程，因此可以在原进程中进行一些需要隔离 namespace 的操作。目前 Docker 没有调用这个 api，其原因在下文中将会介绍。

## Namespace 隔离的实现方法

### UTS(UNIX Time-sharing System)

Docker 中，每个镜像基本都以自身提供的服务来命名镜像的 hostname，其不会对宿主机产生任何影响，其原理就是利用了 UTS namespace：

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)
static char child_stack[STACK_SIZE];
char *const child_args[] = {"/bin/bash", NULL};
// 子进程
int child_main(void *args)
{
    printf("在子进程中、n");
    // sethostname 将主机名设置为参数 1 的值，参数 2 代表名字的字节数
    sethostname("NewNamespace", 12);
    // exec 可以执行用户命令，常使用"/bin/bash"并接受参数，运行起一个 bash
    execv(child_args[0], child_args);
    return 1;
}
// 父进程
int main()
{
    printf("程序开始、n");
    // 创建一个子进程及新的 UTS namespace
    // 若未指定 SIGCHLD 信号，则在子进程终止时不会通知父进程
    int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
    // waitpid 阻塞父进程直到子进程结束
    waitpid(child_pid, NULL, 0);
    printf("已退出、n");
    return 0;
}
```

编译运行后发现我们在进入子进程的同时，主机名也发生了改变，即进入了一个新创建的 UTS namespace。当我们在子进程中的终端输入 exit 后，子进程便调用 execv() 方法结束进程。于是父进程中的 waitpid() 方法停止阻塞，父进程也随之退出。

```bash
[root@koktlzz home]# ./a.out
程序开始
在子进程中
[root@NewNamespace home]# exit
exit
已退出
[root@koktlzz home]# 
```

### IPC(Inter-Process Communication)

两个不同 IPC namespace 下的进程互相不可见，因此也就实现了进程间通信的隔离。其实现方法只需要修改 clone() 方法中的 flag：

```c
int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWIPC | SIGCHLD, NULL);
```

我们可以通过 IPC 资源之一的消息队列来验证 IPC namespace 的隔离，首先创建并查看在当前 namespace 下的所有消息队列：

```c
[root@koktlzz home]# ipcmk -Q
消息队列 id：0
[root@koktlzz home]# ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息      
0x3ca3eb5d 0          root       644        0            0           
```

然后我们编译运行修改后的程序，再次查看消息队列：

```bash
[root@koktlzz home]# ./a.out
程序开始
在子进程中
[root@koktlzz home]# ipcs -q

--------- 消息队列 -----------
键        msqid      拥有者  权限     已用字节数 消息      

[root@koktlzz home]# exit
exit
已退出
```

此时可以发现该 IPC namespace 下没有刚刚创建的消息队列，因此也就实现了进程间通信的隔离。

### PID

PID namespace 的隔离会将进程的 pid 重新分配，这样两个不同 namespace 下的进程的 pid 即使相同也不会崩溃。内核为所有的 PID namespace 维护了一个进程树，最顶端是系统默认创建的 root namespace。每个被创建的新 PID namespace 成为这个进程树的子节点，而创建者 namespace 就是它的父节点。父节点可以看到子节点中的进程，并且可以通过信号等方式对子节点中的进程产生影响；反之，子节点却无法看到父节点 PID namespace 中的任何内容。其实现方法依然只需要修改 clone() 方法中的 flag：

```c
int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWPID | SIGCHLD, NULL);
```

编译运行后即可看到 PID namespace 隔离的效果：

```bash
[root@koktlzz home]# ./a.out
程序开始
在子进程中
[root@koktlzz home]# echo $$
1
[root@koktlzz home]# exit
exit
已退出
[root@koktlzz home]# echo $$
2615550
```

值得注意的是，即使进入了新的 PID namespace，使用 **ps -ef** 命令依然可以看到所有父进程的 pid。这是因为我们还未实现文件系统挂载点的隔离，而该命令本质调用的是真实系统中的/proc 文件内容，看到的自然是所有的进程。

调用 unshare() 和 setns() 方法的进程并不会进入新的 PID namespace 中，否则该进程的 PID 发生变化将会导致一些程序（如调用 getpid() 方法）的崩溃。考虑到这一因素，Docker 的 exec 命令不仅调用 setns() 加入已存在的 namespace，还是会调用 clone() 方法创建一个新的进程。

### Mount

进程在创建 Mount namespace 时，会把当前文件结构复制给新的 namespace。新 namespace 中所有的 mount 操作都只影响自身的文件系统，对外界不会产生任何影响。而挂载传播则定义了挂载对象之间的关系，包括共享关系和从属关系：

- 共享关系：一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然；
- 从属关系：一个挂载对象中的挂载事件会传播到另一个挂载对象，反之不行。

其实现方法依然只需要修改 clone() 方法中的 flag，这里加入 CLONE_NEWPID 参数的目的是方便进行验证：

```c
int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
```

编译并运行程序后，我们发现在重新挂载了/proc 文件 (**mount -t proc proc /proc**) 后的 namespace 中使用 **ps -ef** 命令后就只能看到子进程的 pid 了，并且没有影响到父进程中的/proc 文件挂载。

```bash
[root@koktlzz home]# ./a.out
程序开始
在子进程中
[root@koktlzz home]# mount -t proc proc /proc
[root@koktlzz home]# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 12:39 pts/1    00:00:00 /bin/bash
root          17       1  0 12:40 pts/1    00:00:00 ps -ef
[root@koktlzz home]# exit
exit
已退出
```

### Network

Network namespace 提供了网络资源的隔离，包括网络设备、IPv4 和 IPv6 协议栈、IP 路由表、防火墙、socket 等。
修改 clone() 方法中的 flag 参数：

```c
int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWNET | SIGCHLD, NULL);
```

编译运行程序后我们发现：在新创建的 Network namespace 中，无法通过 DNS 解析域名，甚至连网卡 eth0 都没有了。这说明网络资源已经完全隔离。

```bash
[root@koktlzz home]# gcc hello.c && ./a.out
程序开始
在子进程中
[root@koktlzz home]# ping www.baidu.com
ping: www.baidu.com: 未知的名称或服务
[root@koktlzz home]# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[root@koktlzz home]# exit
exit
已退出
[root@koktlzz home]# ping www.baidu.com
PING www.a.shifen.com (180.101.49.12) 56(84) bytes of data.
64 bytes from 180.101.49.12 (180.101.49.12): icmp_seq=1 ttl=49 time=9.96 ms
64 bytes from 180.101.49.12 (180.101.49.12): icmp_seq=2 ttl=49 time=9.93 ms
```

Docker 采用 CNM(Container Network Model) 实现 network namespace 隔离，CNM 主要由三个部分构成：

- 沙盒（sanbox）：network namespace
- 端点（endpoint）：veth-pair
- 网络（network）：linux bridge 或 vlan

对于 Docker 来说，linux bridge 便是 docker0。veth 相当于网桥上的端口，它工作在链路层不需要配置 IP。而 docker0 自身的 IP 默认为 172.17.0.1/16，即容器的默认网关地址。所有容器都会在 docker0 的子网范围内选取一个未占用的 ip 使用，并通过 veth-pair 连接到 docker0。

![20210131140833](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210131140833.png)

Docker daemon 创建容器网络的过程如下：

- 首先由 Docker daemon 负责创建一个虚拟网络对 veth-pair。一端绑定到 docker0 网桥上，另一端接入容器中。
- 容器内部的初始化进程（即 init 进程）在管道一端循环等待，直到 Docker daemon 从管道另一端向其传输关于 veth-pair 设备的信息，随后关闭管道。
- 最后，init 进程结束等待，启动它的虚拟网卡"eth0"。

### User

一个普通用户的进程通过 clone() 方法创建的子进程可以在这个新的 User namespace 中拥有不同的用户和用户组，这意味着容器内部的 root 用户可能在宿主机上只是一个普通用户，从而为容器提高了极大的自由。User namespace 与 PID namespace 类似，同样是一个层层嵌套的树状结构。

其实现方法依然是在 flag 参数中加入 CLONE_NEWUSER 创建一个新的 User namespace（）:

```c
int child_pid = clone(child_main, child_stack + STACK_SIZE, CLONE_NEWUSER | SIGCHLD, NULL);
```

编译运行后我们可以发现，在创建的新 User namespace 中，user 和 group 变成了 nobody，uid 和 gid 也随之改变。

```bash
[root@koktlzz home]# ./a.out
程序开始
在子进程中
[nobody@koktlzz home]$ id
uid=65534(nobody) gid=65534(nobody) 组=65534(nobody)
[nobody@koktlzz home]$ exit
exit
已退出
[root@koktlzz home]# id
uid=0(root) gid=0(root) 组=0(root)
```

除此之外，我们还要把子节点 namespace 中的初始 user 与其父节点 namespace 中的某个用户建立映射关系。这样子节点 User namespace 想要给父节点 User namespace 发送一个信号或操作某一个文件时，系统就会判断子节点中的用户在父节点中是否有对应的权限。这是通过在 **/proc/[pid]/uid_map**和 **/proc/[pid]/gid_map**文件中写入对应的映射信息实现的，以 uid_map 为例：

```c
void set_uid_map(pid_t pid, int inside_id, int outside_id, int length)
{
    char path[256];
    sprintf(path, "/proc/%d/uid_map", getpid());
    FILE *uid_map = fopen(path, "w");
    // inside_id 表示新建的 User namespace 中对应的 uid/gid
    // outside_id 表示 namespace 外部映射的 uid/gid
    // length 表示映射范围，通常为 1，表示只映射一个
    fprintf(uid_map, "%d %d %d", inside_id, outside_id, length);
    fclose(uid_map);
}
```
