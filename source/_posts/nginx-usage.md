---
title: Nginx反向代理与负载均衡
date: 2024-08-16 14:04:51
categories:
- Web-Server
tags:
- Nginx
---

Nginx常用命令，反向代理与负载均衡实现方法，以及HTTPS配置。

<!--more-->

## 前言

Nginx是高性能的HTTP和反向代理的web服务器，处理高并发能力是十分强大，能支持高达50,000个并发连接数。Nginx支持热部署，启动简单，可以做到7*24不间断运行，几个月都不需要重新启动。Nginx适用于各种场景，包括静态文件服务、反向代理、负载均衡等。

## 基本概念

### 正向/反向代理

正向代理代理的是客户端，而且客户端是知道目标的，而目标是不知道客户端是通过何种方式访问的。

{% asset_img forward_proxy.png 正向代理架构图 %}

反向代理代理的是服务端，客户端对代理是无感知的，客户端不需要任何配置就可以访问。我们只需要把请求发送给反向代理服务器，由反向代理服务器去选择目标服务器获取数据后，再返回给客户端。此时反向代理服务器和目标服务器对外就是一个服务器，暴露的是代理服务器地址，隐藏了真实服务器的地址。

{% asset_img reverse_proxy.png 反向代理架构图 %}

### proxy_pass

proxy_pass是Nginx中一个非常重要的指令，用于将请求代理到后端服务器。它可以用于HTTP和Stream模块，分别处理HTTP请求和TCP/UDP流量。

proxy_pass的基本语法如下，

```
proxy_pass URL;
```

在proxy_pass后面的URL以斜杠/结束，表示绝对根路径。例如下面的例子中，访问`http://example.com/proxy/test.html`会被转发到`http://127.0.0.1/test.html`。

```
server {
    listen 80;
    server_name example.com;

    location /proxy/ {
        proxy_pass http://127.0.0.1/;
    }
}
```

在proxy_pass后面的URL不以斜杠/结束，表示相对路径。例如下面的例子中，访问`http://example.com/proxy/test.html`会被转发到`http://127.0.0.1/proxy/test.html`。

```
server {
    listen 80;
    server_name example.com;

    location /proxy/ {
        proxy_pass http://127.0.0.1
    }
}
```


### 负载均衡

负载均衡是指增加服务器的数量，然后将请求分发到各个服务器上，将原先请求集中到单个服务器上的情况改为将请求分发到多个服务器上。

Nignx提供了三种负载均衡的方式：轮询法(默认)、加权轮询、ip_hash。

这三种负载均衡方式可以组合使用，例如搭建三个服务器并完成反向代理，对应的配置文件如下

```
...
http {
    ...
    upstream backend {
        ip_hash;
        server 127.0.0.1:8000 weight=3;
        server 127.0.0.1:8001;
        server 127.0.0.1:8002;
    }
    ...
    server{
        ...
        location /app {
            proxy_pass http://backend; 
        }
        ...
    }
    ...
}
...
```

## 安装方法

Nignx可通过包管理器、C语言编译、docker等方式安装。

Linux平台下，使用包管理器安装

`sudo apt update`

`sudo apt install nginx`

使用docker安装

`docker pull nginx`

## 常用命令

启动nginx

`nginx`

查看nginx进程

`ps -ef|grep nginx`

查看nginx端口占用情况

`lsof -i:[port]`

停止nginx

`nginx -s stop` 或 `nginx -s quit`

重载配置文件

`nginx reload`

重新打开配置文件

`nginx reopen`

查看安装目录、编译参数、日志文件及配置文件位置

`nginx -V`

查看nginx配置文件

`nginx -t`

静态文件部署

`cp -rf * [path]`

## HTTPS配置

HTTPS协议需要使用SSL证书，在主流的云平台上都可以申请到免费的SSL证书，也可以通过openssl命令生成一个自签名的证书(未经过CA认证)。

### 使用openssl生成证书

生成私钥文件(private key)

`openssl genrsa -out private.key 2048`

根据私钥生成证书签名请求文件(csr文件)

`openssl req -new -key private.key -out cert.csr`

使用私钥对证书申请进行签名从而生成证书文件(pem文件)

`openssl x509 -req -in cert.csr -out cacert.pem -signkey private.key`

### 修改nginx配置

```
...
http {
    ...
    server{
        listen   443 ssl;
        server_name         localhost;
        return 301 https://$server_name$request_uri;
        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ssl_session_timeout 5m;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        ...
    }
    ...
}
...
```

## 参考文档

- [Nignx官方文档](http://nginx.org/en/docs/)