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

.dockerignore是Docker镜像构建的ignore文件，通过合理配置可以减少Docker构建上下文大小，加快镜像构建速度，避免将敏感信息（如开发环境配置）打包进镜像，保持生产镜像的简洁性。

## Docker Compose

Docker Compose是一个用来定义和运行复杂应用的Docker工具。一个使用Docker容器的应用，通常由多个容器组成。使用Docker Compose不再需要使用shell脚本来启动容器。 

Docker Compose通过一个配置文件来管理多个Docker容器。在配置文件中，所有的容器通过services来定义，然后使用docker-compose脚本来启动，停止和重启应用、应用中的服务以及所有依赖服务的容器，非常适合组合使用多个容器进行开发的场景。

最新的Docker已经集成了docker-compose功能，可使用`docker compose version`命令查看当前版本。

### 简单示例

下面是一个Docker Compose的简单示例。

#### 配置文件目录结构

```
├─ etc
   └─ docker
       ├─ mysql
       │   ├─ config
       │   │    └─ my.conf
       │   ├─ sqlScript
       │   │    └─ createDatabase.sql
       │   └─ Dockerfile
       │
       └─ redis
            └─ conf
                └─ redis.conf 
```

#### 配置文件docker-compose.yml

Docker Compose配置文件是一个定义服务，网络和卷的YAML文件，默认文件名为docker-compose.yml。与Docker运行一样，默认情况下，Dockerfile中指定的选项（例如，CMD，EXPOSE，VOLUME，ENV）都被遵守，你不需要在docker-compose.yml中再次指定它们。

例如定义一个包含redis和mysql数据的容器，可使用以下配置文件。

```yml
# server.yml
version: '3.7'

services:

  redis:
    container_name: redis # 指定一个自定义容器名称，而不是生成的默认名称 
    ports: # 端口信息。常用的简单格式：使用宿主:容器(HOST:CONTAINER)格式或者仅仅指定容器的端口(宿主将会随机选择端口)都可以。
      - "6379:6379"
    image: redis # 指定启动容器的镜像，可以是镜像仓库/标签或者镜像id
    volumes: # 卷挂载路径设置。可以设置宿主机路径(HOST:CONTAINER)或加上访问模式(HOST:CONTAINER:ro),挂载数据卷的默认权限是读写(rw)，可以通过ro指定为只读。
      - "./redis/datadir:/data" # 相对于当前compose文件的相对路径
      - "./redis/conf/redis.conf:/usr/local/etc/redis/redis.conf"
      - "./redis/logs:/logs"
    command: # 覆盖容器启动后默认执行的命令
      /bin/bash -c "redis-server /usr/local/etc/redis/redis.conf"      
    
  mysql-db:
    container_name: mysql-db 
    ports:
      - "3306:3306"
    image: mysql:8.0.1                   
    volumes: # 包含命名卷(如mysql_data:/var/lib/mysql)、主机路径卷、匿名卷、只读卷和文件挂载(如./init.sql:/docker-entrypoint-initdb.d/init.sql)等 
      - "./mysql/data:/var/lib/mysql"           
      - "./mysql/config:/etc/mysql/conf.d"     
    build: # 指定包含构建上下文的路径
      context: . # 包含Dockerfile文件的目录路径，或者是git仓库的URL。当提供的值是相对路径时，它被解释为相对于当前compose文件的位置。该目录也是发送到Docker守护程序构建镜像的上下文。
      dockerfile: mysql/Dockerfile # 备用Docker文件。Compose将使用备用文件来构建，还必须指定构建路径。
    environment: # 添加环境变量。可以使用数组或字典两种形式。只给定名称的变量会自动获取它在Compose主机上的值，可以用来防止泄露不必要的数据。(如果服务指定了build选项，那么在构建过程中通过environment定义的环境变量将不会起作用，将使用build的args子选项来定义构建时的环境变量。)
      MYSQL_ROOT_PASSWORD: "root"
```

可使用以下命令构建并运行容器(使用-f使用自定义的配置文件，使用-d以后台方式运行)

`docker-compose -f server.yml up -d`

关闭或移除容器、镜像等

`docker-compose -f server.yml down`

### 完整示例

{% asset_img docker_compose_sample.png Docker-Compose示例 %}

将一个.NET+Vue+MySQL应用打包为Docker镜像的完整方案如下，

```
my-app/
├── mysql
│   ├─ config
│   │    └─ my.conf
│   ├─ data
│   ├─ sqlScript
│   │    └─ init.sql
│   └─ Dockerfile
│
├── backend/
│   ├── ABPDemo.Web
│   │    └── ABPDemo.Web.csproj
│   ├── Dockerfile
│   ├── .dockerignore
│   └── ...  
│              
├── frontend/
│   ├── package.json
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── nginx.conf
│   └── ...
├── docker-compose.yml
└── .env
```

#### 配置数据库Dockerfile

数据库Dockerfile文件如下，

```dockerfile
FROM mysql:8.0.1
COPY mysql/sqlScript/*.sql /docker-entrypoint-initdb.d/
```

其中init.sql数据库初始化脚本文件如下，

```sql
-- init.sql
-- MySQL 数据库初始化脚本

-- 设置字符集
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- 创建数据库（如果不存在）
CREATE DATABASE IF NOT EXISTS `ABPDemo` 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;

-- 使用新创建的数据库
USE `ABPDemo`;

-- 示例: 创建学生表（如果不存在）
CREATE TABLE IF NOT EXISTS `students` (
	`id` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '数据唯一标识',
	`name` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '姓名',
	PRIMARY KEY (`id`) USING BTREE,
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=DYNAMIC;

INSERT INTO students (id, name) VALUES('3a1b6256-17fb-3551-0aed-af1436e871f1', 'John');

-- 启用外键约束
SET FOREIGN_KEY_CHECKS = 1;
```

my.conf数据库配置文件如下，

```ini
[mysqld]
character-set-server=utf8mb4
default-time-zone='+8:00'
innodb_rollback_on_timeout='ON'
max_connections=500
innodb_lock_wait_timeout=500
```

注：如果MySQL容器初始化脚本未执行，可使用命令`cmd.exe /c "docker exec -i mysql_db mysql -uroot -p[MYSQL_ROOT_PASSWORD] < ./mysql/sqlScript/init.sql"`手动执行。

#### 配置后端Dockerfile

.NET后端Dockerfile文件如下，

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
# 复制解决方案文件（如果存在）
COPY *.sln .
# 复制项目文件进行依赖恢复
COPY ["ABPDemo.Web/ABPDemo.Web.csproj", "ABPDemo.Web/"]
COPY ["ABPDemo.HttpApi/ABPDemo.HttpApi.csproj", "ABPDemo.HttpApi/"]
COPY ["ABPDemo.Application.Contracts/ABPDemo.Application.Contracts.csproj", "ABPDemo.Application.Contracts/"]
COPY ["ABPDemo.Domain.Shared/ABPDemo.Domain.Shared.csproj", "ABPDemo.Domain.Shared/"]
COPY ["ABPDemo.Application/ABPDemo.Application.csproj", "ABPDemo.Application/"]
COPY ["ABPDemo.Domain/ABPDemo.Domain.csproj", "ABPDemo.Domain/"]
COPY ["ABPDemo.EntityFrameworkCore/ABPDemo.EntityFrameworkCore.csproj", "ABPDemo.EntityFrameworkCore/"]
RUN dotnet restore "ABPDemo.Web/ABPDemo.Web.csproj"
# 复制所有源代码
COPY . .
WORKDIR "/src/ABPDemo.Web"
RUN dotnet build "ABPDemo.Web.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "ABPDemo.Web.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "ABPDemo.Web.dll"]
```

#### 配置前端Dockerfile

Vue前端Dockerfile文件如下，

```dockerfile
# 构建阶段
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
# 覆写.env.production中的值
ARG VUE_APP_API_BASE_URL=http://localhost:8081/
ENV VUE_APP_BASE_URL=$VUE_APP_API_BASE_URL
RUN npm run build:prod

# 生产阶段
FROM nginx:1.27.0
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

其中nginx.conf的配置内容如下，

```
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        # 处理前端路由
        location / {
            try_files $uri $uri/ /index.html;
        }

        # 代理 API 请求到后端
        location /api/ {
            proxy_pass http://backend:80;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

注：Dockerfile中配置的后端地址`http://localhost:8081/`为代理地址(由于浏览器同源策略，必须与宿主机的前端访问地址一致)，由于宿主机的8081端口指向容器的80端口，因此容器内实际的前端地址仍然为`http://localhost:80/`。 在location中通过设置proxy_pass相对路径`http://backend:80`(Docker内置的DNS解析器会自动将backend解析为对应容器的IP地址)，将后端请求由`http://localhost:80/api/xxx`转发到`http://backend:80/api/xxx`，从而解决跨域问题。

#### 配置Docker Compose

docker-compose.yml配置文件如下，

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0.1
    container_name: mysql_db
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3307:3306"
    volumes:
      - ./mysql/data:/var/lib/mysql #mysql数据存储
      - ./mysql/config:/etc/mysql/conf.d   #mysql的配置
      - ./mysql/sqlScript:/docker-entrypoint-initdb.d #mysql初始化脚本  
    networks:
      - app-network
    #healthcheck:
    #  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
    #  timeout: 20s
    #  retries: 10
    command: 
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --ssl=0 # 禁用 SSL

  backend:
    build: ./backend
    container_name: dotnet_backend
    environment:
      # 使用"SectionName__PropertyName"替换appsettings.json中的数据库连接字符串
      # 例如：
      # "ConnectionStrings": {
      #   "DefaultConnection": "Server=127.0.0.1;port=3306;Database=ABPDemo;User=root;Password=root1234"
      # }
      # 对应的数据库连接字符串替换如下(mysql是docker-compose中定义的MySQL服务名称，Docker内置的DNS解析器会自动将服务名解析为对应容器的IP地址):
      - ConnectionStrings__DefaultConnection=Server=mysql;Port=3306;Database=${MYSQL_DATABASE};User=${MYSQL_USER};Password=${MYSQL_PASSWORD}
      - ASPNETCORE_ENVIRONMENT=Production
    ports:
      - "5000:80"
    depends_on:
      #mysql:
      #  condition: service_healthy
      - mysql
    networks:
      - app-network
    restart: unless-stopped

  frontend:
    build: ./frontend
    container_name: vue_frontend
    ports:
      - "8081:80"
    depends_on:
      - backend
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mysql_data:

networks:
  app-network:
    driver: bridge
```

#### 配置环境变量

.env配置文件如下，

```
MYSQL_ROOT_PASSWORD=root1234
MYSQL_DATABASE=ABPDemo
MYSQL_USER=root
MYSQL_PASSWORD=root1234
```

使用命令`docker-compose build`以构建上述所有镜像，使用命令`docker-compose up -d`启动镜像。

#### 导入导出

管道方式打包导出所有镜像(使用此命令导出的镜像，在加载后为`<none>:<none>`未标签状态，需要手动打标签)
`docker save -o all-images.tar $(docker-compose images -q)`

列表方式打包导出所有镜像(列表项格式为`<image-name>:<image-tag>`)
`docker save -o all-images.tar mysql:8.0.1 my-app-backend:latest my-app-frontend:latest`

也可以将所有镜像罗列到image-list.txt当中，然后通过文件读取的方式导出
`docker save -o all-images.tar $(cat image-list.txt)`

> 注：使用列表导出时，请检查所有Dockfile中依赖的基础镜像，例如node:18-alpine等，将他们全部添加进列表中，否则后续docker-compose构建容器时会尝试联网拉取远程基础镜像。

导出配置
`docker-compose config > docker-compose-export.yml`

加载镜像
`docker load -i all-images.tar`

根据导出配置启动服务
`docker-compose -f docker-compose-export.yml up -d`


## 常用命令

### 镜像管理

拉取镜像

`docker pull [image-url]`

使用镜像源拉取(如轩辕镜像中mirror-url为`docker.xuanyuan.me`, path为`library`)

`docker pull [mirror-url]/[path]/[image-name]:[image-version]`

从本地归档文件(.tar)加载镜像到本地镜像库

`docker load -i [image-name].tar`

通过Shell重定向使用标准输入(stdin)加载镜像(适用于脚本或管道操作)

`docker load < [image-name].tar`

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

保存为镜像文件

`docker save -o [image-name].tar [image-name]`

### 容器管理

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

退出容器

`exit`

## WSL配置Docker

通过WSL使用Docker时可能会缺失一些配置文件。

### daemon.json

daemon.json是Docker引擎的配置管理文件，可以统一设置容器的网络、存储、安全、日志等选项。

docker安装后默认没有daemon.json这个配置文件，需要进行手动创建。在linux系统中，配置文件的默认径为：/etc/docker/daemon.json。

可使用以下命令手动创建daemon.json

`sudo mkdir -p /etc/docker`

`sudo touch /etc/docker/daemon.json`

`sudo nano /etc/docker/daemon.json`

例如配置docker镜像，可添加以下配置

```json
{
    "registry-mirrors": [ 
      "https://<your-aliyun-id>.mirror.aliyuncs.com",
      "https://docker.xuanyuan.me" 
    ]
}
```

保存并退出后可使用以下命令重启docker

`sudo systemctl restart docker`

可使用`docker info`查看镜像配置是否生效，若仍未生效，可前往Docker Desktop手动设置。

右键点击任务栏Docker图标 → Settings → Docker Engine，修改JSON配置后点击Apply & Restart等待重启完成即可。

### docker.service

docker.service文件是用于配置和管理Docker守护进程的systemd单元文件。它定义了Docker服务的启动、停止和重启行为，以及一些关键的配置参数。

如果缺失这个文件，在重启docker时会报错

> Failed to start docker.service: Unit docker.service not found

可使用以下命令手动创建docker.service

`sudo nano /etc/systemd/system/docker.service`

可添加以下配置

```
[Unit]
Description=Docker Application Container Engine
After=network-online.target

[Service]
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
Restart=always

[Install]
WantedBy=multi-user.target
```

保存后使用以下命令重新加载systemd配置并启动服务

`sudo systemctl daemon-reload`

`sudo systemctl start docker`

## 参考文档

- [Docker官方文档](https://docs.docker.com/)

- [Docker Compose官方文档](https://docs.docker.com/compose/)

- [如何在Docker中通过环境变量覆写数据库连接字符串](https://kontext.tech/project/aspnet-core/article/overwrite-connecting-string-via-environment-variable-for-asp-net-core-on-docker)

- [Docker集成WSL2](https://docs.docker.com/desktop/features/wsl/)