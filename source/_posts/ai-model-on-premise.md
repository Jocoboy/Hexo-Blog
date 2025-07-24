---
title: AI大模型本地部署
date: 2025-07-17 13:52:26
categories:
- Artificial-Intelligence
tags:
- Ollama
- MaxKB
- RAG
---

使用大模型部署和管理工具Ollama在本地部署AI大模型，并使用基于大语言模型和RAG的知识库问答系统MaxKB为大模型添加UI界面。

<!--more-->

## 前言

本地部署AI模型可以确保数据隐私和安全，降低长期成本，并提供更高的性能和控制权。本地部署AI模型的方案有很多，例如使用桌面应用程序LMstudio、使用大模型命令行部署和管理工具Ollama等。

本地大模型部署完成后，通常还需要添加UI界面，可以使用MaxKB、OpenWebUI等。二者侧重点不同，MaxKB侧重于为大模型添加关联知识库，OpenWebUI侧重于大模型的使用和管理。

## Ollama

Ollama是一个命令行大模型部署和管理工具，可以用它将类似deepseek-r1这样的AI大模型部署到本地。Ollama支持运行Llama 3、Phi 3、Mistral、Gemma和其他模型，并允许定制和创建自己的模型。

### 安装使用

Ollama支持MacOS、Linux、Windows等多种操作系统，在[Ollama官网](https://ollama.com/)下载安装Ollama后，输入地址`http://localhost:11434/`可查看Ollama是否成功运行。

可以使用`ollama run [model-name]`命令下载并运行AI模型，下载完成后可以使用命令行进行对话，对话响应速度和显卡性能有关，输入`>>> /bye`可以结束当前对话。使用`ollama list`命令可查看当前已下载的AI模型。

### Rest API方式交互

除了命令行交互的方式，Ollama同样支持REST API的方式管理和运行大模型。

例如`POST /api/generate`使用提供的模型为给定提示生成响应，对应curl命令

```
curl http://localhost:11434/api/generate -d '{
  "model": "deepseek-r1",
  "prompt": "Why is the sky blue?"
}'
```

## MaxKB

MaxKB是一个基于大语言模型和RAG的知识库问答系统。

### RAG

检索增强生成RAG(Retrieval-Augmented Generation)是一种结合了信息检索技术与语言生成模型的人工智能技术。

#### 基本原理

由于大模型都是基于预训练的数据集训练出来的，如果你的问题超出了这个数据集的范围，那么它生成的内容很可能是不够准确的，也称为大模型的幻觉。

为了解决这个问题，通常需要将检索模型和生成模型结合在一起，通过从外部数据源添加一些上下文和背景知识，确保大模型有足够的信息来生成更加准确和可靠的答案。

#### 工作流程

{% asset_img rag_workflow.png RAG工作流程 %}

首先用户提出问题后，这个问题会被作为一个查询条件，从你的向量化知识库中进行检索，找到最相关的信息或文档，然后这些信息和文档会随着问题一起输入到大模型中，最后大模型会根据这些信息来生成答案，确保生成的答案更加准确和可靠。

### 安装和部署

由于MaxKB只支持Linux系统(Unbuntu和CentOS等)上使用，在Windows上使用需要借助Docker。

可以使用以下命令下载MaxKB镜像到本地，创建并启动一个MaxKB容器。

`docker run -d --name=maxkb -p 8088:8080 -v ~/.maxkb:/var/lib/postgresql/data 1panel/maxkb`

启动后输入网址`http://localhost:8088`即可访问，默认账号为`admin`，密码为`MaxKB@123..`。

登录后点击模型设置，选择Ollama并点击添加模型，填写模型相关信息后会自动下载大模型，其中API URL填写`http://host.docker.internal:11434`。添加完成后点击应用，点击设置填写相关应用信息即可完成创建。

> 注：host.docker.internal是Docker提供的一个特殊DNS名称，指向宿主机(Host)的内部IP地址。当容器需要访问宿主机上运行的服务(如数据库、API或其他应用)时，可以使用这个地址。

## ngrok

为了将AI大模型应用分享给其他人，需要使用ngrok。ngrok是一款强大的内网穿透工具，可帮助用户将本地服务暴露到公网，实现在外网访问本地服务。

### 安装配置

在[ngrok官网](https://ngrok.com/)下载安装ngrok，下载完成后登录ngrok个人账号，复制AUTHTOKEN，并执行`ngrok config add-authtoken $YOUR_AUTHTOKEN`命令即可完成配置。

配置完成后即可使用ngrok映射大模型应用，输入`ngrok http 8088`将会回显一个Forwarding地址，这个地址就是ngrok为我们映射的公网地址，将其替换`http://localhost:8088`即可。

注：对于所有HTTP请求，ngrok默认会返回一个html的警告页面，可在上述命令中添加参数`--request-header-add ngrok-skip-browser-warning:true`跳过此警告。

除此之外，还可以通过修改`%HOMEPATH%\AppData\Local\ngrok`目录下的ngrok.yml配置文件来实现。

```yml
version: "3"
agent:
    authtoken: 'your-authtoken'
tunnels:
  example:
    proto: http
    addr: 8088
    request_header:
      add: ["ngrok-skip-browser-warning: true"]
```

修改完成后使用命令`ngrok start --all`即可启动服务。

## 参考文档

- [Ollama API文档参考](https://github.com/ollama/ollama/blob/main/docs/api.md)

- [MaxKB官方文档](https://maxkb.cn/docs/v2/installation/offline_installtion/)

- [ngrok官方文档](https://ngrok.com/docs)
