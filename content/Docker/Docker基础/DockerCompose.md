---
title: "Docker Compose"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  docker:
    parent: "Docker 基础"
weight: 300
---

## 简介

学习了 [Docker 基础知识](/docker/docker基础/狂神docker课程笔记/) 后，我们已经可以使用 Dockerfile 和 **docker build** 命令创建一个镜像，并使用 **docker run** 命令运行一个容器。但如果想要同时运行多个容器，并建立容器之间的依赖关系，仅仅依靠上述的命令就显得十分复杂。因此，我们需要一个新的工具能够高效地对多个容器进行运行管理（批量容器编排），这便是 Docker Compose。

[官方文档：](https://docs.docker.com/compose/)
> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration. To learn more about all the features of Compose, see [the list of features](https://docs.docker.com/compose/#features).
> Compose works in all environments: production, staging, development, testing, as well as CI workflows. You can learn more about each case in [Common Use Cases](https://docs.docker.com/compose/#common-use-cases).
> Using Compose is basically a three-step process:
>
> 1. Define your app’s environment with a **Dockerfile** so it can be reproduced anywhere.
> 2. Define the services that make up your app in **docker-compose.yml** so they can be run together in an isolated environment.
> 3. Run **docker-compose up** and Compose starts and runs your entire app.
>
## 安装

### 下载

```bash
# 通过项目源地址下载
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
# 国内镜像地址
https://get.daocloud.io/docker/compose/releases/download/1.25.5/docker-compose
```

### 修改可执行权限

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

### 安装成功确认

```bash
docker-compose --version
```

## Getting Started

### 官方文档

> [https://docs.docker.com/compose/gettingstarted/](https://docs.docker.com/compose/gettingstarted/)

### 环境配置

- 创建项目目录

  ```bash
  mkdir composetest
  cd composetest
  ```

- 创建一个 python 应用 app.py，并将下列代码复制进去：

  ```python
  import time
  
  import redis
  from flask import Flask
  
  app = Flask(__name__)
  cache = redis.Redis(host='redis', port=6379)
  
  def get_hit_count():
      retries = 5
      while True:
          try:
              return cache.incr('hits')
          except redis.exceptions.ConnectionError as exc:
              if retries == 0:
                  raise exc
              retries -= 1
              time.sleep(0.5)
  
  @app.route('/')
  def hello():
      count = get_hit_count()
      return 'Hello World! I have been seen {} times.\n'.format(count)
  ```

- 创建一个文本文档 requirements.txt，并添加相应依赖

    ```bash
    flask
    redis
    ```

### 创建 Dockerfile 文件

```bash
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY . .
CMD ["flask", "run"]
```

Dcokerfile 文件中每一行地作用已在相应章节有过阐述，官方文档也对此进行了详细说明：

> - Build an image starting with the Python 3.7 image.
> - Set the working directory to **/code**.
> - Set environment variables used by the **flask** command.
> - Install gcc and other dependencies
> - Copy **requirements.txt** and install the Python dependencies.
> - Add metadata to the image to describe that the container is listening on port 5000
> - Copy the current directory `.` in the project to the workdir `.` in the image.
> - Set the default command for the container to **flask run**.

### 定义服务

在项目目录下，建立一个 docker-compose.yml 文件，并添加如下代码：

```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
  redis:
    image: "redis:alpine"
```

该文件定义了两个服务：web 和 redis。其中 web 服务使用 Dockerfile 和 **build .** 命令在当前文件夹下构建，redis 服务则直接使用官方镜像"redis:alpine"。

### 构建并运行应用

在当前文件夹下运行命令 **docker-compose up**，系统开始根据我们写好的 Dockerfile 构建了一个新镜像，并下载了一个 redis 镜像。这时出现了一个 **Warning**：

```bash
WARNING: Image for service web was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Pulling redis (redis:alpine)...
```

根据提示重新构建镜像：

```bash
docker-compose build
```

当提示构建成功之后，我们再次运行命令 **docker-compose up**：

![20201213171656](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213171656.png)

可以发现应用已经重新启动了。如果我们在另一个终端中使用 **docker ps** 命令，即可看到正在运行的两个服务：

![20201213171715](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213171715.png)

也可以使用 **docker images** 命令查看到新增的镜像：

![20201213171921](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213171921.png)

还可以使用 **docker network ls** 命令查看到新增的网络 **composetest_default**：

![20201213171948](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213171948.png)

接着使用 **docker network inspect** 命令查看该网络的详细信息，发现 web 和 redis 服务均在该网络下运行。因此一个应用下的多个服务均可以通过容器名访问而无需使用 ip 地址：

![20201213172013](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213172013.png)

![20201213172201](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213172201.png)

另外，在 app.py 中我们已经定义了 redis 服务的服务名：

```python
cache = redis.Redis(host='redis', port=6379)
```

因此通过服务名来访问 redis 服务也是可以的：

![20201213172039](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213172039.png)

最后，我们只要打开 web 服务暴露的端口 5000，即可看到 app.py 脚本的实现：

![20201213172059](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213172059.png)

服务的名称（composetest_web_1 和 composetest_redis_1 ）是有其命名规则的，第一项是我们的项目目录 composetest，第二项是具体的服务名（web 和 redis），第三项是一个编号。由于我们是单机启动，编号自然为 1。而如果是在一个集群中同时运行这项服务，那么编号便可以将这个集群中的不同服务器上运行的服务加以区分。

### 为服务添加挂载

修改我们之前创建的 yml 文件：

```yaml
version: "3.9"
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_ENV: development
  redis:
    image: "redis:alpine"
```

**volume** 键值对使得项目目录与 web 服务的工作目录/code 进行挂载，因此我们可以即时修改代码而不用重建镜像。另外，**environment** 键规定了 flask 在开发模式下运行，并在更改后重载代码。
随后使用命令 **docker-compose up** 重启应用，这时如果我们修改项目目录里的 app.py 中的输出语句：

```python
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
```

就会发现 **localhost:5000** 显示的网页发生了改变：

![20201213172300](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20201213172300.png)

## yml 文件编写规则

官方文档：
> [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)

配置文件的编写总共分为三层：

1. 版本

    ```yaml
    version: "3.x"
    ```

2. 服务

    ```yaml
    services: #服务
    
      redis:  # 服务 1
        # 服务配置与 docker 容器配置相同
        image: redis:alpine
        ports:
          - "6379"
        networks:
          - frontend
      ......
      db:  # 服务 2
        image: postgres:9.4
        volumes:
          - db-data:/var/lib/postgresql/data
        networks:
          - backend
        ......
    ```

3. 其他配置

    ```yaml
    networks:   # 网络
      frontend:
      backend:
    
    volumes:    # 挂载
      db-data:
    configs:    # 配置
      my_config:
        file: ./my_config.txt    
    ......
    ```

## 部署 Wordpress 博客

官方文档：
> [https://docs.docker.com/compose/wordpress/](https://docs.docker.com/compose/wordpress/)
