---
title: "狂神 Docker 课程笔记"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  docker:
    parent: "Docker 基础"
weight: 200
---
## 课程链接

> [https://www.bilibili.com/video/BV1og4y1q7M4?from=search&seid=18106350651153543104](https://www.bilibili.com/video/BV1og4y1q7M4?from=search&seid=18106350651153543104)

## Docker 为什么出现？

- 开发和运维两套环境，而环境配置十分麻烦。
  如在 Windows 上开发，要发布到 Linux 上运行。Docker 给以上问题提出了解决方案：
  Java --- Jar（环境）---打包项目带上环境（镜像）---Docker 仓库（应用商店）---下载镜像---直接运行
- Docker 的思想来自于集装箱，核心思想：隔离。
  即将应用打包装箱，每个箱子是互相隔离的，可以将服务器利用到极致。
- [官方文档](https://docs.docker.com/)
- [仓库地址](https://hub.docker.com/)

## Docker 能做什么？

### 传统虚拟机与 Docker 对比

![20201209124302](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201209124302.png)

### Docker 的优点

- 不模拟完整的操作系统，系统内核（kernel）非常小，更少的抽象层（GuestOS：如 Centos）
- 容器内的应用直接运行在宿主机的内核，容器本身没有自己的内核，也没有虚拟硬件。
- 每个容器相互隔离，内部都有属于自己的文件系统，互不影响。

### Docker 实现 DevOps

- 应用更快速的交付和部署
  打包镜像发布测试，一键运行；不再需要写大量帮助文档，安装程序
- 更便捷的升级和扩缩容？
  部署应用就和搭积木一样
- 更简单的系统运维
  开发和测试的环境高度一致
- 更高效的计算资源利用
  内核级别的虚拟化，可以在一个物理机上运行很多的容器实例，服务器性能可以被压榨到极致。

## Docker 的基本组成

![20201208171750](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208171750.png)

- 镜像（image）：镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件。它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。相当于一个模板，通过这个模板来创建容器服务，可以通过一个镜像创建多个容器。
- 容器（container）：独立运行一个或一组应用/基本命令有：启动，停止，删除等。可理解为一个简单的 linux 系统。
- 仓库（repository）：存放镜像的地方（公有/私有）

## Docker 运行原理

Docker 是一个 Client-Server 结构的系统，以守护进程运行在主机上。通过 Socket 从客户端进行访问。

## Docker 的常用命令

![20201212210110](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212210110.png)

### 帮助命令

```bash
docker --help            # 帮助信息
docker info              # 系统信息，包括镜像和容器的数量
```

帮助文档地址：[https://docs.docker.com/engine/reference/](https://docs.docker.com/engine/reference/)

### 镜像命令

```bash
docker search 镜像名      # 搜索镜像
docker pull 镜像名        # 下载镜像
```

Docker 采用联合文件系统，不同镜像的相同文件无需再次下载：

![20201212192925](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212192925.png)

```bash
docker rmi 镜像名/id      删除镜像
```

### 容器命令

```bash
docker run [options] 镜像名/id [command]  # 建立容器并启动：         
[options]:                
            --name=容器名                 # 命名容器以区分不同容器
            -d                           # 在后台运行容器（必须有一个前台进程，否则进程会自动关闭）
            -it                          # 使用交互方式运行，进入容器查看内容
            -p 主机端口：容器端口            # 暴露指定容器端口
            -P                           # 暴露容器所有端口
[command]:
            /bin/bash                    # 控制台
```

![2](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/2.jpg)

```bash
Exit                         # 从容器中退回主机 
CTRL+Q+P                     # 容器不停止退出
docker ps                    # 显示当前运行的容器 
          -a                 # 带出历史运行过的容器
docker rm 容器名/id           # 删除指定容器
docker rm $(docker ps -aq)   # 删除全部容器
```

### 其他命令

```bash
docker start/restart/stop/kill 容器名/id             
docker logs -tf --tail 显示的日志条数 容器名/id  # 查看日志
docker top 容器名/id                 # 查看容器中的进程信息
docker inspect 容器名/id             # 查看镜像的元数据
docker exec -it 容器名/id /bin/bash  # 通常容器以后台方式运行，需要进入其中修改配置：进入容器后开启一个新终端       
docker attach 容器名/id              # 进入容器正在执行的终端
docker cp 容器名/id: 容器内路径 主机文件路径       # 从容器内拷贝文件到主机上
```

## Docker 镜像详解

![133](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208182908.png)

### UnionFS（联合文件系统）

- 联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。
- 特性：一次同时加载多个文件系统，但从外面看起来只能看到一个文件系统。联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

### 镜像加载原理

Docker 的镜像实际由一层一层的文件系统组成：

- bootfs（boot file system）主要包含 bootloader 和 kernel。bootloader 主要是引导加载 kernel，完成后整个内核就都在内存中了。此时内存的使用权已由 bootfs 转交给内核，系统卸载 bootfs。可以被不同的 Linux 发行版公用。
- rootfs（root file system），包含典型 Linux 系统中的/dev，/proc，/bin，/etc 等标准目录和文件。rootfs 就是各种不同操作系统发行版（Ubuntu，Centos 等）。因为底层直接用 Host 的 kernel，rootfs 只包含最基本的命令，工具和程序就可以了。
- 分层理解：所有的 Docker 镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的容器层。
容器在启动时会在镜像最外层上建立一层可读写的容器层（R/W），而镜像层是只读的（R/O）。
  
![20201208183559](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208183559.png)

```bash
docker commit -m="描述信息" -a="作者" 容器 id 目标镜像名：[tag]  # 编辑容器后提交容器成为一个新镜像
```

## 容器数据卷

![20201208183759](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208183759.png)

### 什么是容器数据卷？

为了实现数据持久化，使容器之间可以共享数据。可以将容器内的目录，挂载到宿主机上或其他容器内，实现同步和共享的操作。即使将容器删除，挂载到本地的数据卷也不会丢失。

### 使用容器数据卷

使用命令：

```bash
dokcer run -it -v 主机内目录：容器内目录 镜像名/id
```

将容器内目录挂载到主机内目录上，通过 **docker inspect** 命令查看该容器即可以看到挂载信息：

![20201211162429](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211162429.png)

建立挂载关系后，只要使用命令在主机内新建一个文件：

```bash
touch /home/mountdir/test.txt
```

就会在容器内的挂载目录下发现相同的文件（test.txt），从而实现了容器和主机的文件同步和共享：

![20201211162933](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211162933.png)

### 匿名挂载

```bash
docker run -d  -v 容器内目录  镜像名/id  # 匿名挂载
```

匿名挂载后，使用 **docker volume ls** 命令查看所有挂载的卷：

![20201208184201](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208184201.png)

每一个 VOLUME NAME 对应一个挂载的卷，由于挂载时未指定主机目录，因此无法直接找到目录。

### 具名挂载

```bash
docker run -d  -v 卷名：容器内目录  镜像名/id  # 具名挂载
```

![20201211164244](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211164244.png)

可以发现挂载的卷：volume01，并通过 **docker volume inspect 卷名** 命令找到主机内目录：

![20201211164505](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211164505.png)

所有 docker 容器内的卷，在未指定主机内目录时，都在：*/var/lib/docker/volumes/卷名/_data* 下，可通过具名挂载可以方便的找到卷，因此广泛使用这种方式进行挂载。

### 数据卷容器

![20201208184546](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208184546.png)

```bash
docker run -it --name container02 --volumes from container01 镜像名/id  # 将两个容器进行挂载
```

## DockerFile

Dockerfile 是用来构建 docker 镜像的文件

### 构建步骤

编写一个 dockerfile 文件，随后运行命令：

```bash
docker build -f 文件路径 -t 标签 .  # 文件名为 Dockerfile 时可省略且最后的。不要忽略
docker run     # 运行镜像
docker push    # 发布镜像
```

### dockerfile 命令

| 命令 | 效果 |
| - | - |
| FROM | 基础镜像：Centos/Ubuntu |
| MAINTAINER | 镜像作者+邮箱 |
| RUN | 镜像构建的时候需要运行的命令 |
| ADD | 为镜像添加内容，可以是文件或 URL 资源。若为压缩包则会解压缩 |
| WORKDIR | 镜像工作目录（进入容器时的目录） |
| VOLUME | 挂载的目录 |
| EXPOSE | 暴露端口配置 |
| CMD/ENTRYPOINT | 指定这个容器启动时要运行的命令 |
| COPY | 类似于 ADD。将文件拷贝到镜像中，但不会进行解压缩操作，也不能指定 URL 资源 |
| ENV | 构建时设置环境变量 |

Dockerfile 应最多包含一条 ENTRYPOINT 命令和一条 CMD 命令，如：

```bash
ENTRYPOINT ["/bin/ping"]
CMD ["localhost"]
```

如果编写了多条命令，则只有最后一条命令生效。由于默认的 ENTRYPOINT 命令是 **/bin/sh -c**，因此可以在不指定 ENTRYPOINT 的前提下运行 CMD，如：

```bash
CMD ["/bin/ping", "localhost"]
```

另外在启动容器时，**docker run \<image\>** 后可以添加参数覆盖 CMD 命令，从而增加容器的灵活性。因此如果使用 **docker run \<image\> baidu.com** 命令启动容器，那么在上述第一个例子中容器将会 ping baidu.com 而非 localhost。

### 构建过程

- 每个保留关键字（指令）都必须是大写字母
- 从上到下顺序执行
- "#" 表示注释
- 每一个指令都会创建提交一个新的镜像层并提交

### 构建实例

![20201208185616](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201208185616.png)

## Docker 网络

### 理解 Doker0

通过命令 **ip addr** 查看本地 ip 地址，我们发现除了本机回环地址和埃里远的内网地址外，还多了一个网卡：Docker0，这是 Docker 服务启动后自动生成的。

![20201211165741](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211165741.png)

而如果进入一个正在后台运行的 tomcat 容器，同样使用 **ip addr** 命令，发现容器得到了一个新的网络：**12: eth@if13**，ip 地址：**172.17.0.2**。这是 Docker 在容器启动时为其分配的。

![1607676753(1)](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/1607676753(1).png)

思考一个问题：此时我们的 linux 主机可以 ping 通容器内部（**172.17.0.2**）吗？（**注意与容器暴露端口相区分**）

![20201211170424](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211170424.png)

linux 可以 ping 通 docker 容器内部，因为 docker0 的 ip 地址为 **172.17.0.1**，容器为 **172.17.0.2**。原理：我们每启动一个 docker 容器，docker 就会给容器分配一个默认的可用 ip，我们只要安装了 docker，就会有一个网卡 docker0(bridge)。网卡采用桥接模式，并使用 veth-pair 技术（veth-pair 就是一堆虚拟设备接口，成对出现，一段连着协议，一段彼此相连，充当一个桥梁。）。
这时我们退出容器，回到主机再次观察主机的 ip 地址：

![20201211170810](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20201211170810.png)

我们惊奇地发现了一个新网络 **13: vethda1df4b@if12**，对应容器内网络地址的 **12: eth@if13**。
容器和容器之间是可以互相 ping 通的：容器 1→Docker0→容器 2

![20210131140833](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210131140833.png)

docker 中的所有网络接口都是虚拟的 ，转发效率高。删除容器后，对应的网桥也随之删除。

### --link

若编写一个微服务并连接数据库，如果数据库 ip 改变，如何根据容器名而不是 ip 访问容器？显然，直接使用容器名是无法 ping 通容器内部的：

![20201212200850](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212200850.png)

这时我们可以在容器启动命令中加入一个选项：**--link**，使得我们可以根据容器名来访问容器。

```bash
docker run -d -P --link 容器名/id 镜像名/id
```

![20201212200748](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212200748.png)

然而反向就不可以 ping 通，这是因为--link 的本质是把需要连接的容器名/id 写入启动容器的配置文件中，即增加了一个 ip 和容器名/id 的映射：

![20201212201930](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212201930.png)

目前已经不建议使用这种方式。

### 自定义网络

我们使用命令：

```bash
docker network ls    # 查看所有的 docker 网络
```

![20201212202126](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212202126.png)

docker 中的网络模式有：

- bridge：桥接（docker 默认）/
- none：不配置网络 /
- host：和宿主机共享网络

**docker run** 命令默认带有一个参数--net bridge，此处的 bridge 指的就是 docker0。如果我们不想使用 docker0，那如何创建一个新的网络呢？

```bash
docker  network create --driver 网络模式 --subnet 子网 ip --gateway 网关 网络名       
```

![20201212205640](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212205640.png)

我们不仅在 **docker network ls** 命令下发现了这个新创建的网络 newnet，还可以使用 **docker network inspect** 命令查看其详细信息，包括了我们创建时定义的子网 ip 和网关：

![20201212202733](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212202733.png)

只要两个容器启动时都通过 **--net**，选用了同一个已创建的网络，不同容器间即可通过 ip 地址或容器名/id 连通：

![20201212203023](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212203023.png)

### 网络连通

![20201212204434](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212204434.png)

对于建立在不同网络下 (docker0, newnet) 的两个容器 tomcat01 和 tomcat02，他们的网段不同，因此是无法彼此 ping 通容器内部的：

![20201212204311](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212204311.png)

这时我们需要通过 **docker network connect** 命令打通容器与网络之间的连接：

```bash
docker network connect 网络名 容器名/id
```

![20201212203803](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212203803.png)

这个功能类似于将一个容器赋予多个 ip 地址，同样可以用 **docker network inspect** 命令查看网络连通后，该网络的变化：

![20201212204235](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201212204235.png)

原本 newnet 网络中只含有 tomcat02，现在增加了 tomcat01，因此可以连通。
