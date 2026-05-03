# 后端服务通信：HTTP、RPC、专用协议、Socket、TCP/IP

核心先记一句：

> HTTP 只是应用层协议的一种，不是所有通信都叫 HTTP。

不同对象之间通信，使用的应用层协议不同：

```text
前端 ──HTTP──> 后端服务
                 │
                 ├──HTTP / gRPC──────> 用户服务 / 支付服务 / 库存服务
                 │
                 ├──MySQL 协议───────> MySQL
                 │
                 ├──PostgreSQL 协议──> PostgreSQL
                 │
                 ├──Redis 协议───────> Redis
                 │
                 └──Kafka 协议───────> Kafka
```

也就是：

```text
后端应用服务 ↔ 后端应用服务
常用 HTTP / gRPC

后端应用服务 ↔ MySQL
用 MySQL 协议

后端应用服务 ↔ PostgreSQL
用 PostgreSQL 协议

后端应用服务 ↔ Redis
用 Redis 协议

后端应用服务 ↔ Kafka
用 Kafka 协议
```

前端调后端常用 HTTP。

后端调另一个后端，也常用 HTTP 或 gRPC。

但后端调数据库、Redis、Kafka，通常用它们自己的专用协议。

所以不能把“网络通信”都理解成 HTTP。HTTP 只是其中一种应用层协议。MySQL、PostgreSQL、Redis、Kafka 也都有自己的通信协议。

---

## 1. 应用层协议不同，但底层大多还是 socket + TCP/IP

虽然上层协议不同：

```text
HTTP
gRPC
MySQL 协议
PostgreSQL 协议
Redis 协议
Kafka 协议
```

但它们底层大多还是：

```text
socket + TCP/IP
```

可以理解成：

```text
应用层：决定双方“怎么说话”
        │
        ├──HTTP：前端和后端常用
        ├──gRPC：后端服务之间常用
        ├──MySQL 协议：后端访问 MySQL
        ├──PostgreSQL 协议：后端访问 PostgreSQL
        ├──Redis 协议：后端访问 Redis
        └──Kafka 协议：后端访问 Kafka
        │
        v
传输层 / 网络层：负责把数据送过去
        │
        v
socket + TCP/IP
```

所以完整理解应该是：

```text
不是：
所有通信 = HTTP

而是：
不同服务使用不同应用层协议
        │
        v
这些协议的数据最终大多通过 socket 接入 TCP/IP 网络
        │
        v
再传到目标服务
```

---

## 2. TCP 三次握手的位置

TCP 三次握手不是 HTTP 请求本身。

它发生在真正传业务数据之前。

也就是说：

```text
先建立 TCP 连接
        │
        v
再传应用层数据
```

应用层数据可以是：

```text
HTTP 请求
MySQL 查询
Redis 命令
Kafka 消息
gRPC 调用数据
```

TCP 三次握手的过程是：

```text
第一步：客户端发送 SYN
表示：我想建立连接

第二步：服务端返回 SYN + ACK
表示：我收到了你的连接请求，也同意建立连接

第三步：客户端返回 ACK
表示：我也收到了你的确认
```

三步完成后，TCP 连接才算建立成功。

然后才开始传真正的业务数据。

```text
TCP 三次握手建立连接
        │
        v
TCP 连接建立成功
        │
        v
开始传 HTTP 请求 / MySQL 查询 / Redis 命令 / Kafka 消息 / gRPC 调用数据
```

以前端请求后端为例：

```text
三次握手建立 TCP 连接
        │
        v
浏览器发送 HTTP 请求：GET /api/order/1
        │
        v
后端返回 HTTP 响应：订单数据
```

所以：

```text
TCP 三次握手解决的是：
连接能不能建立

HTTP / MySQL / Redis / Kafka / gRPC 解决的是：
连接建立后，双方按照什么格式交流业务数据
```

这才是关键分层。

---

## 3. 后端服务之间除了 HTTP，还有 RPC

后端服务之间不只有 HTTP。

微服务之间还常用：

```text
gRPC
Dubbo
Thrift
自定义 TCP 协议
消息队列
```

其中 gRPC 是很常见的一种 RPC 方式。

RPC 可以先理解成：

```text
像调用本地函数一样调用远程服务
```

比如订单服务要调用库存服务：

```text
订单服务
  │
  │ gRPC
  v
库存服务
```

概念上像：

```text
stockService.decrease(product_id, count)
```

而不是手写：

```text
POST /stock/decrease
```

HTTP 的感觉是：

```text
我向某个 URL 发请求
```

RPC 的感觉是：

```text
我调用远程服务里的某个函数
```

gRPC 底层常基于 HTTP/2，但使用方式更像远程函数调用。

所以这句话要压住：

```text
后端调另一个后端，可以用 HTTP，也可以用 RPC。
gRPC 是 RPC 的一种常见实现。
它底层常基于 HTTP/2，但使用方式更像调用远程函数。
```

---

## 4. 最终压缩成一条完整链路

把所有内容压成一条线，就是：

```text
程序想访问另一个服务
        │
        v
先通过 socket 接入网络
        │
        v
如果走 TCP，先进行 TCP 三次握手
        │
        v
TCP 连接建立成功
        │
        v
开始传应用层数据
        │
        ├──前端调后端：HTTP 请求 / 响应
        ├──后端调后端：HTTP / gRPC / Dubbo / Thrift / 自定义 TCP 协议
        ├──后端调 MySQL：MySQL 协议
        ├──后端调 PostgreSQL：PostgreSQL 协议
        ├──后端调 Redis：Redis 协议
        └──后端调 Kafka：Kafka 协议
```

再压缩一点：

```text
socket + TCP/IP 是底层通信基础
应用层协议决定具体交流格式

前端和后端常用 HTTP
后端和后端常用 HTTP / gRPC
后端和数据库、Redis、Kafka 通常用各自专用协议

TCP 三次握手发生在传业务数据之前
连接建立后，才开始传 HTTP 请求、MySQL 查询、Redis 命令、Kafka 消息或 gRPC 调用数据
```

---

## 5. 最短记忆版

```text
HTTP 不是全部。

前端调后端：
HTTP

后端调后端：
HTTP / gRPC / RPC / 消息队列

后端调 MySQL：
MySQL 协议

后端调 PostgreSQL：
PostgreSQL 协议

后端调 Redis：
Redis 协议

后端调 Kafka：
Kafka 协议

底层大多：
socket + TCP/IP

TCP 三次握手：
发生在传业务数据之前，用来建立连接

连接建立后：
才开始传 HTTP 请求、数据库查询、Redis 命令、Kafka 消息、gRPC 调用数据
```
