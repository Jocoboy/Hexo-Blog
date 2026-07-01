---
title: kubernetes核心组件与集群搭建
date: 2024-08-17 13:18:34
categories:
- Architecture
tags:
- Kubernetes
---

kubernetes的核心组件、架构体系、环境搭建(minikube/k3s)，以及kubectl常用命令和可视化管理工具Portainer。

<!--more-->

## 前言

kubernetes是一个全新的基于容器技术的分布式架构领先方案。kubernetes的本质是一组服务器集群，它可以在集群的每个节点上运行特定的程序，来对节点中的容器进行管理。目的是实现资源管理的自动化。

## 基本概念

### 核心组件

{% asset_img kubernetes_components.png kubernetes部分核心组件 %}

- node，一个物理机或虚拟机
- pod，一个或多个应用容器的组合(如sidecar模式，应用容器与辅助容器共用一个Pod)。pod之间可通过内部ip地址访问
- svc(service)，将一组pod封装为一个服务，可以通过一个统一的入口来访问，相当于反向代理，可以解决因pod销毁导致的内部ip地址变更问题
- node:port，端口节点，提供内部服务对外的IP映射
- ing(ingress)，用来管理从集群外部访问集群内部服务的入口和方式，可以配置不同的转发规则，根据不同的规则访问不同的svc，以及svc对应的pod。此外还可以配置域名、负载均衡、SSL证书等
- cm(config map)，用来存储应用程序配置信息，实现应用程序与配置信息的解耦
- secret，为cm中的敏感信息提供Base64加密，需要配合其他安全机制(如网络控制、访问控制、身份认证)一起使用
- vol(volume)，可以将应用数据挂载到集群内部的本地磁盘上，或是集群外部的远程存储上，实现数据的持久化
- deploy(deployment)，定义和管理应用程序的副本数量，是pod上的一层抽象
- replicaset，介于pod与deploy之间，用于管理pod
- sts(stateful set)，和deploy类似，用来管理有状态的应用(如数据库、缓存、消息队列等)

## Master-Worker架构

kubernetes是典型的Master-Worker架构。Master-Node负责管理整个集群，Worker-Node负责运行应用程序和服务。

{% asset_img kubernetes_architecture.png kubernetes架构图 %}

- kubelet，负责管理和维护每个node上的pod
- kube-proxy，负责提供网络代理和负载均衡服务
- container-runtime，负责提供容器运行时(如docker engine)
- kube-apiserver，负责提供API接口服务
- etcd，高可用的键值存储系统，负责存储集群中各种资源对象的状态信息
- c-m(controller manager)，负责管理集群中各种资源对象的状态
- sched(schedular)，负责监控集群中所有节点的资源使用情况，然后根据一些调度策略，将pod调度到合适的node上运行
- c-c-m(cloud controller manager)，云平台控制器，负责与云平台的api交互

## 环境搭建

### minikube

minukube是一个轻量级的kubernetes实现，可在本地计算机上创建虚拟机，并部署仅包含一个节点的简单集群。

kubectl是一个命令行工具，可以通过在命令行输入各种命令与MasterNode的kube-apiserver交互，从而与Kubernetes集群进行交互。

在windows中使用chocolatey安装minukube

`choco install minkube`

查看版本信息验证是否安装成功

`minikube version`

创建集群(选择国内镜像源并指定版本)

`minikube start --image-mirror-country='cn'  --kubernetes-version=v1.23.9`

查看集群中的节点信息

`kubectl get nodes`

Docker Desktop也自带了minukube，可手动开启。

### k3s

k3s是一个CNCF认证的轻量级的kubernetes发行版，可以方便地搭建一个多节点集群。

#### 准备虚拟机环境

在windows中使用chocolatey安装multipass

`choco install multipass`

创建一个名为k3s的虚拟机

`multipass launch --name k3s`

启动虚拟机

`multipass start k3s`

登录到虚拟机

`multipass shell k3s`

通过multipass创建的虚拟机默认不允许SSH远程登录，需要额外配置。

- 添加用户密码 `sudo passwd ubuntu`
- 修改SSH配置 `sudo vi /etc/ssh/sshd_config`
    ```
    PubkeyAuthentication yes
    PasswordAuthentication yes
    KbdInteractiveAuthentication yes    
    ```
- 重启SSH服务 `sudo service ssh restart`
- 使用SSH登录 `ssh ubuntu@ip`

#### 创建和配置Master节点

使用国内镜像安装k3s

`curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -`

查看当前节点(Master节点)

`sudo kubectl get nodes`

#### 创建和配置Worker节点

在Master节点上获取token，作为其它节点加入集群的凭证

`sudo cat /var/lib/rancher/k3s/server/node-token`

将TOKEN保存到环境变量

`TOKEN=$(multipass exec k3s sudo cat /var/lib/rancher/k3s/server/node-token)`

保存master节点的IP地址

`MASTER_IP=$(multipass info k3s | grep IPv4 | awk '{print $2}')`

使用刚刚的TOKEN和MASTER_IP来创建两个worker节点，并把它们加入到集群中

- 创建两个worker节点的虚拟机

    `multipass launch --name worker1 --cpus 2 --memory 8G --disk 10G`

    `multipass launch --name worker2 --cpus 2 --memory 8G --disk 10G`

- 在worker节点虚拟机上安装k3s

    `for f in 1 2; do
        multipass exec worker$f -- bash -c "curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=\"https://$MASTER_IP:6443\" K3S_TOKEN=\"$TOKEN\" sh -"
    done`

### 在线环境

k8s也可通过在线环境使用

- [labs.play-with-k8s](https://labs.play-with-k8s.com/)

- [killercoda](https://killercoda.com/)

## kubectl常用命令

创建一个pod

`sudo kubectl run [pod-name] --image=[image-name]`

创建一个deployment，指定镜像为nginx

`sudo kubectl create deployment [deployment-name] --image=[image-name]`

通过配置文件创建一个deployment

`vi [deployment-name].yaml`

`sudo kubectl create -f [deployment-name].yaml`

修改deployment

`sudo kubectl edit deployment [deployment-name]`

通过配置文件修改deployment

`sudo kubectl apply -f [deployment-name].yaml`

删除deployment

`sudo kubectl delete deployment [deployment-name]`

通过配置文件删除deployment

`sudo kubectl delete -f [deployment-name].yaml`

查看pod或deployment

`sudo kubectl get pod`

`sudo kubectl get deployment`

查看pod日志

`sudo kubectl logs [pod-name]`

将deployment对外公开为service

`sudo kubectl expose deployment [deployment-name]`

查看服务详细信息

`sudo kubectl describe service [deployment-name]`

删除服务

`sudo kubectl delete service [deployment-name]`

通过配置文件创建NodePort类型的服务

```yaml
apiVersion: v1  # 使用的Kubernetes API版本，这里是v1。
kind: Service  # 定义资源类型为Service，表示创建一个服务。
metadata:  # 元数据部分，用于描述Service的基本信息。
  name: nginx-service  # Service的名称为nginx-service。
spec:  # 规格部分，定义Service的规格。
  type: NodePort	# 指定服务类型，默认为ClusterIP
  selector:  # 选择器部分，用于指定服务应该选择哪些Pod作为后端。
    app: nginx  # 选择具有标签app=nginx的Pod作为后端。
  ports:  # 端口配置，定义Service暴露的端口。
    - protocol: TCP  # 使用TCP协议。
      port: 80  # Service暴露的端口号为80。
      targetPort: 80  # 转发到后端Pod的端口号也为80。
      nodePort: 30080
```

`sudo kubectl apply -f [service-name].yaml`

查看当前集群命名空间

`sudo kubectl ns`

## 可视化管理工具Portainer

安装portainer，并将其暴露在NodePort=30777上

`kubectl apply -n portainer -f https://downloads.portainer.io/ce2-19/portainer.yaml`

在集群外部输入ip地址+端口号即可访问。

## 参考文档

- [kubernetes官方文档](https://kubernetes.io/docs/home/)

- [minukube官方文档](https://minikube.sigs.k8s.io/docs/)

- [multipass官方文档-常用命令](https://multipass.run/docs/multipass-cli-client)

- [k3s官方文档-快速入门](https://docs.rancher.cn/docs/k3s/quick-start/_index/)

- [Portainer官方文档](https://docs.portainer.io/)