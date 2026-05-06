# M11.1 Docker 部署 Neo4j 服务五要素

> 日期：2026-05-06  
> 主题：Docker Compose 部署 Neo4j 时，如何理解 image、container、environment、ports、volumes

---

## 0. ASCII 结构目录

```text
Docker Compose 部署一个服务
│
├── 1. image 镜像
│   └── 决定“跑什么服务”
│       └── 例如：neo4j:5.14-community
│
├── 2. container 容器
│   └── 镜像运行起来后的隔离环境
│       └── 例如：legal-agent-neo4j
│
├── 3. environment 环境变量
│   └── 决定“服务怎么配置”
│       └── 例如：账号、密码、内存参数
│
├── 4. ports 端口映射
│   └── 决定“外部怎么访问容器里的服务”
│       ├── 7474:7474  → Neo4j Browser
│       └── 7687:7687  → Bolt Driver
│
├── 5. volumes 数据卷
│   └── 决定“容器里的数据怎么保存到外面”
│       ├── neo4j_data:/data
│       └── neo4j_logs:/logs
│
└── 最终理解
    ├── 镜像决定跑什么
    ├── 容器负责跑起来
    ├── 环境变量决定怎么配置
    ├── 端口决定怎么访问
    └── 数据卷决定怎么保存
```

---

## 1. 一句话总结

Docker Compose 部署一个服务，本质就是：

```text
用镜像启动容器，
用环境变量配置容器，
用端口映射访问容器，
用数据卷保存容器数据。
```

对应到 Neo4j：

```text
image：用 Neo4j 镜像
container：启动 Neo4j 容器
environment：配置账号密码和内存
ports：暴露 Neo4j 的访问端口
volumes：保存 Neo4j 的数据和日志
```

---

## 2. image：镜像决定跑什么服务

镜像可以理解为：

```text
提前打包好的服务模板。
```

比如：

```yaml
image: neo4j:5.14-community
```

意思是：

```text
使用 Neo4j 5.14 community 这个镜像模板。
```

镜像里面已经准备好了：

```text
Neo4j 程序
运行依赖
默认目录结构
启动方式
```

需要注意的是：

```text
一个镜像通常打包一个主要服务。
```

比如：

```text
neo4j 镜像 → 主要跑 Neo4j
mysql 镜像 → 主要跑 MySQL
redis 镜像 → 主要跑 Redis
```

不是“一个镜像里面有很多种服务”。  
多个服务是由 `docker-compose.yml` 组织起来的。

例如：

```text
postgres 服务
redis 服务
neo4j 服务
```

最终理解：

```text
镜像是单个服务的模板。
docker-compose 是把多个服务组合起来启动。
```

---

## 3. container：容器是镜像跑起来后的隔离环境

容器可以理解成：

```text
镜像运行起来之后的实例。
```

比如：

```yaml
container_name: legal-agent-neo4j
```

意思是：

```text
把 neo4j:5.14-community 这个镜像跑起来，
运行出来的容器叫 legal-agent-neo4j。
```

容器里面可以有：

```text
程序
依赖
配置
运行目录
环境变量
端口
数据挂载
```

比如 Neo4j 容器里有：

```text
Neo4j 程序
Java 运行环境
Neo4j 配置
/data 数据目录
/logs 日志目录
7474 端口
7687 端口
```

镜像和容器的关系：

```text
镜像 = 模板 / 安装包
容器 = 跑起来的实例
```

程序员可以用这个类比记：

```text
镜像像 class
容器像 object
```

---

## 4. 容器和虚拟机的区别

容器可以粗略理解成“小虚拟机”，但它不是完整虚拟机。

虚拟机是：

```text
一台完整的虚拟电脑。
```

它里面有：

```text
完整操作系统
内核
CPU / 内存 / 磁盘虚拟化
自己的网络
自己的文件系统
```

容器不是这样。

容器是：

```text
共享宿主机操作系统内核，
只隔离进程、文件系统、网络、环境变量。
```

所以更准确地说：

```text
虚拟机 = 模拟一台完整电脑
容器 = 隔离一个程序运行环境
```

类比：

```text
虚拟机像：在电脑里又装了一台完整小电脑。
容器像：在同一台电脑里隔出一个独立房间。
```

所以面试时不要把容器直接说成“虚拟机”。更稳的表达是：

```text
容器不是完整虚拟机，而是基于宿主机内核的轻量级隔离运行环境。
```

---

## 5. environment：环境变量决定服务怎么配置

环境变量是在容器启动时传进去的配置。

例如：

```yaml
environment:
  NEO4J_AUTH: neo4j/legal_agent_dev_pwd
  NEO4J_dbms_memory_pagecache_size: 512M
  NEO4J_dbms_memory_heap_max__size: 1G
```

它的意思是：

```text
配置 Neo4j 的账号密码和内存参数。
```

其中：

```text
NEO4J_AUTH: neo4j/legal_agent_dev_pwd
```

表示：

```text
用户名：neo4j
密码：legal_agent_dev_pwd
```

所以 environment 的作用是：

```text
服务启动前，把必要配置传给容器里的程序。
```

最短记忆：

```text
environment 决定服务怎么配置。
```

---

## 6. ports：端口映射决定外部怎么访问容器

容器默认是隔离的。

Neo4j 在容器里运行时，会监听自己的内部端口：

```text
7474
7687
```

但宿主机不能直接访问容器内部端口，所以要做端口映射：

```yaml
ports:
  - "7474:7474"
  - "7687:7687"
```

格式是：

```text
宿主机端口:容器内部端口
```

也就是：

```text
宿主机 7474 → 容器内部 7474
宿主机 7687 → 容器内部 7687
```

含义：

```text
7474：Neo4j Browser 页面端口
7687：Neo4j Bolt 协议端口，Python 程序连接 Neo4j 用
```

所以：

```text
访问 http://localhost:7474
= 访问容器里的 Neo4j Browser

连接 bolt://localhost:7687
= Python 程序连接容器里的 Neo4j 服务
```

如果没有端口映射，Neo4j 虽然在容器里正常运行，但宿主机和外部程序访问不到它。服务住进屋里，门却没开，多少有点人类工程味道。

最短记忆：

```text
ports 决定外面怎么访问容器里的服务。
```

---

## 7. volumes：数据卷决定数据怎么保存

容器可以删除。

如果数据只存在容器内部：

```text
容器删除
↓
数据也没了
```

所以要配置数据卷：

```yaml
volumes:
  - neo4j_data:/data
  - neo4j_logs:/logs
```

意思是：

```text
容器内部 /data → Docker volume neo4j_data
容器内部 /logs → Docker volume neo4j_logs
```

其中：

```text
/data：保存 Neo4j 图数据库数据
/logs：保存 Neo4j 日志
```

这样即使容器被删除：

```text
容器删了
↓
数据卷还在
↓
重新启动容器还能继续使用原来的图谱数据
```

所以 volumes 的作用是：

```text
把容器里的关键数据保存到容器外面，防止容器删除后数据丢失。
```

最短记忆：

```text
volumes 决定容器里的数据怎么保存到外面。
```

---

## 8. 回到 Neo4j 的 docker-compose 配置

Neo4j 服务配置如下：

```yaml
neo4j:
  image: neo4j:5.14-community
  container_name: legal-agent-neo4j
  restart: unless-stopped
  environment:
    NEO4J_AUTH: neo4j/legal_agent_dev_pwd
    NEO4J_dbms_memory_pagecache_size: 512M
    NEO4J_dbms_memory_heap_max__size: 1G
  ports:
    - "7474:7474"
    - "7687:7687"
  volumes:
    - neo4j_data:/data
    - neo4j_logs:/logs
```

逐行翻译：

```text
image: neo4j:5.14-community
= 使用 Neo4j 5.14 community 镜像

container_name: legal-agent-neo4j
= 启动出来的容器叫 legal-agent-neo4j

restart: unless-stopped
= 除非手动停止，否则容器异常退出后自动重启

environment
= 配置 Neo4j 的账号密码和内存参数

ports:
  7474:7474
  7687:7687
= 把容器里的 Neo4j 页面端口和驱动端口暴露到本机

volumes:
  neo4j_data:/data
  neo4j_logs:/logs
= 把 Neo4j 数据和日志保存到容器外，避免容器删了数据也没了
```

翻译成人话：

```text
用 neo4j:5.14-community 这个镜像，
启动一个叫 legal-agent-neo4j 的容器，
设置账号密码和内存参数，
把容器里的 7474 和 7687 端口暴露到本机，
把容器里的 /data 和 /logs 保存到 Docker 数据卷，
这样 Neo4j 能独立运行、能被访问、数据不会因为容器删除而丢失。
```

---

## 9. 最短记忆版

```text
镜像：程序模板。
容器：镜像跑起来后的隔离环境。
环境变量：启动时传给程序的配置。
端口：让外部访问容器里的服务。
数据卷：让容器删除后数据还能保留。
```

再压缩：

```text
image 决定跑什么服务；
container 是服务跑起来的隔离环境；
environment 决定服务怎么配置；
ports 决定外面怎么访问容器里的服务；
volumes 决定容器里的数据怎么保存到外面。
```

最终总理解：

```text
镜像是单个服务的模板；
容器是这个服务跑起来的实例；
docker-compose 是把多个容器服务组织到一起启动。
```

---

## 10. 面试回复版

如果面试官问：

```text
你是怎么用 Docker 部署 Neo4j 的？
docker-compose 里 image、ports、volumes 分别是什么意思？
容器和虚拟机有什么区别？
```

可以这样回答：

```text
我理解 Docker Compose 部署一个服务，核心就是配置五件事：镜像、容器、环境变量、端口映射和数据卷。

以 Neo4j 为例，image 指定使用 neo4j:5.14-community 这个镜像，镜像里已经打包好了 Neo4j 程序和运行依赖。Docker 会根据这个镜像启动一个容器，container_name 用来指定容器名称，比如 legal-agent-neo4j。

environment 用来配置服务启动参数，比如 Neo4j 的账号密码和内存参数。ports 用来做端口映射，例如 7474:7474 把容器内部的 Neo4j Browser 页面端口暴露到宿主机，7687:7687 把 Bolt 协议端口暴露出来，供 Python 驱动连接。volumes 用来做数据持久化，比如把容器内部的 /data 和 /logs 挂载到 neo4j_data、neo4j_logs 数据卷里，避免容器删除后图数据库数据丢失。

容器可以粗略理解成一个隔离运行环境，但它不是完整虚拟机。虚拟机通常包含完整操作系统和内核，而 Docker 容器共享宿主机内核，只隔离进程、文件系统、网络和环境变量，所以更轻量。

一句话概括就是：镜像决定跑什么，容器负责跑起来，环境变量决定怎么配置，端口决定怎么访问，数据卷决定怎么保存。
```

更短的面试版：

```text
Docker Compose 部署 Neo4j，本质就是用 image 指定 Neo4j 镜像，用 container_name 指定容器名，用 environment 配置账号密码和内存，用 ports 把 7474 和 7687 暴露到宿主机，用 volumes 把 /data 和 /logs 挂载到数据卷，保证容器删除后数据不丢。容器不是完整虚拟机，而是共享宿主机内核的轻量级隔离运行环境。
```
