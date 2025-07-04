---
title: Docker虚拟化技术与容器化部署
date: 2024-08-15 12:05:50
categories:
- Virtualization-Technology
tags:
- Docker
---

Docker基本概念、安装配置、容器化部署及常用命令。

<!--more-->

## 前言

Docker是一种操作系统层面的轻量级虚拟化技术，由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器(Container)。与虚拟机(VM)相比，Docker使用宿主机的操作系统，启动更快。每个容器只运行所需的应用程序和依赖项，资源消耗更少。Docker将操作系统、运行时环境、第三方软件库和依赖包、应用程序、环境变量、配置文件、启动命令等打包在一起，以便在任何环境中都能正常运行。

## 基本概念

{% asset_img docker_architecture.png Docker架构图 %}

### Docker Daemon

Docker使用Client-Server架构，Docker Clinet和Docker Daemon之间通过Socket或Restful API进行通信。

Docker Daemon是服务端的守护进程，负责管理Docker的各种资源，接受并处理来自客户端的请求，然后将结果返回给客户端。

### Docker镜像与容器

镜像(Images)是一个只读的容器模板，含有启动Docker容器所需的文件系统结构及内容。容器是Docker的运行实例，它提供了一个独立的可移植的环境。Docker以镜像和在镜像基础上构建的容器为基础，以容器开发、测试、发布的单元将应用相关的所有组件和环境进行封装，避免了应用在不同平台间迁移所带来的依赖问题，确保了应用在生产环境的各阶段达到高度一致的实际效果。

### Docker仓库

Docker仓库(Registry)是用来集中存储和管理Docker镜像的地方。常用的有Dockerhub，用户可在此分享和下载Docker镜像，以实现镜像的共享和复用。

## 安装配置

Docker官网在国内需要vpn才能访问，可通过国内镜像地址下载。Docker的使用可通过命令行方式，也可通过图形化工具Docker Desktop。

Windows系统中启动Docker Desktop的先决条件(以下二选一)
- 安装WSL(推荐)
- 开启Hyper-V功能

## 容器化与Dockerfile

Dockerfile是Docker用来构建镜像的指令文件，Docker容器化包含以下三个部分

- 创建一个Dockerfile

- 使用Dockerfile构建镜像

- 使用镜像创建和构建容器

例如要使用node在alpine中运行一个index.js文件，对应的Dockerfile为

```dockerfile
FROM node:14-alpine
COPY index.js /index.js
CMD node /index.js
```

### 常用命令

#### 镜像管理

拉取镜像

`docker pull [image-url]`

使用镜像源拉取(如轩辕镜像`docker.xuanyuan.me`)

`docker pull [mirror-url]/library/[image-name]:latest`

从本地归档文件(.tar)加载镜像到本地镜像库

`docker load -i [image-name].tar`

构建镜像

`docker build -t [image-name] .`

根据镜像ID重命名镜像名称

`docker tag [image-id] [image-name]`

运行镜像(使用-d以守护进程/后台方式运行)

`docker run [image-name] .`

查看所有镜像

`docker image ls`

`docker images`

上传镜像

`docker push [image-url]`

删除镜像

`docker rmi [image-name] /`

`docker image rm [image-name]`

从容器创建镜像

`docker commit [container-name] [image-name]`

#### 容器管理

创建容器

`docker create [image-name]`

启动运行并命名容器

`docker run  --name [container-name] [image-name]`

例如运行一个基于Ubuntu的容器，可使用以下命令

`docker run --name ubuntu_demo -itd docker.xuanyuan.me/library/ubuntu`

(注：-itd为组合参数，-i使容器保持交互状态，-t为容器分配一个伪终端，-d在后台运行容器)

进入Ubuntu容器内部

`docker exec -it ubuntu_demo /bin/bash`

停止容器

`docker stop [container-name]`

删除容器

`docker rm [container-name] /`

`docker container rm [container-name]`


## 参考文档

- [Docker官方文档](https://docs.docker.com/)

- [Docker集成WSL2](https://docs.docker.com/desktop/features/wsl/)