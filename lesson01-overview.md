# 第1课：gRPC 全景概览与 RPC 基础

## 学习目标

- 理解 RPC 的演进历程与核心问题
- 掌握 gRPC 的设计哲学与技术栈组成
- 建立 grpc-go 项目结构的全局认知地图
- 能够编译运行示例并观察 gRPC 底层通信

---

## 1.1 RPC 的演进：从本地调用到远程调用

### 什么是 RPC？

RPC（Remote Procedure Call）的核心思想是：**让远程计算机上的函数调用看起来像本地调用一样简单**。

```
本地调用：  result = add(1, 2)         // 直接在进程内执行
RPC调用：   result = stub.add(1, 2)    // 看起来一样，但跨越了网络
```

### RPC 演进时间线

```
1984  Sun RPC / ONC RPC        ─── 最早期的 RPC 框架，NIS/NFS 使用
      │  问题：与特定传输协议绑定，无类型系统
      ▼
1995  DCOM / CORBA             ─── 企业级分布式对象模型
      │  问题：协议复杂，互操作性差，部署沉重
      ▼
2000  XML-RPC → SOAP           ─── 基于 HTTP + XML 的 RPC
      │  问题：XML 冗余大，解析慢，人可读但机器低效
      ▼
2005  REST (JSON over HTTP)    ─── 资源导向，无强类型契约
      │  问题：无代码生成，无流式，手工维护 API 契约
      ▼
2015  gRPC (HTTP/2 + Protobuf) ─── 高性能、强类型、多语言、流式
```

### RPC 必须解决的核心问题

| 问题 | 说明 | gRPC 的解法 |
|------|------|-------------|
| **序列化** | 数据结构 → 字节流 → 数据结构 | Protocol Buffers |
| **传输协议** | 字节流如何在网络上搬运 | HTTP/2 |
| **服务寻址** | 调用哪个服务的哪个方法？ | `/package.service/method` 路径 |
| **连接管理** | 如何建立、复用、关闭连接 | ClientConn + 连接池 |
| **错误处理** | 远程调用失败如何表达？ | gRPC Status Codes |
| **安全认证** | 如何证明身份、加密通信 | TLS + Credentials |

---

## 1.2 gRPC 的诞生背景与设计哲学

### 为什么 Google 要创造 gRPC？

Google 内部有 **数十亿** 的跨服务 RPC 调用/秒，需要：
1. **高性能**：微服务间通信的延迟必须极低
2. **多语言**：后端用 C++/Java/Go/Python，必须统一协议
3. **流式处理**：搜索建议、日志流等需要双向流
4. **强契约**：数千服务、数万 API，手工维护不可能

### gRPC vs REST vs GraphQL

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│     维度      │    gRPC      │    REST      │   GraphQL    │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 传输协议      │ HTTP/2       │ HTTP/1.1     │ HTTP/1.1     │
│ 数据格式      │ Protobuf     │ JSON         │ JSON         │
│ 契约方式      │ .proto 文件   │ OpenAPI      │ Schema SDL   │
│ 代码生成      │ 原生支持      │ 需第三方工具  │ 需第三方工具  │
│ 流式支持      │ 四种模式      │ 无           │ 订阅模式     │
│ 浏览器支持    │ gRPC-Web     │ 原生         │ 原生         │
│ 性能         │ 极高         │ 中等         │ 中等         │
│ 可读性       │ 二进制不可读   │ 人可读       │ 人可读       │
│ 典型场景      │ 微服务间通信  │ 公开 API     │ BFF 聚合层   │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### gRPC 设计哲学

1. **服务优先（Service-First）**：先定义服务接口（.proto），再生成代码
2. **HTTP/2 原生**：充分利用多路复用、流、头部压缩
3. **语言中立**：11+ 语言的官方实现，共享同一套协议规范
4. **可扩展性**：拦截器、自定义 Resolver/Balancer/Codec
5. **面向失败设计**：Deadline 传播、透明重试、健康检查

---

## 1.3 gRPC 核心技术栈

gRPC 不是单一技术，而是三层协议的精密组合：

```
┌─────────────────────────────────────────────────────────┐
│                   你的业务代码                            │
│          Greeter.SayHello(ctx, &HelloRequest{})          │
├─────────────────────────────────────────────────────────┤
│                 gRPC 框架层                              │
│   ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
│   │拦截器    │  │负载均衡   │  │重试/超时  │  │健康检查│  │
│   │Intercept│  │Balancer  │  │Retry/DDL │  │Health  │  │
│   └─────────┘  └──────────┘  └──────────┘  └────────┘  │
│   ┌─────────┐  ┌──────────┐  ┌──────────┐              │
│   │服务发现  │  │Channel   │  │元数据    │              │
│   │Resolver │  │ClientConn│  │Metadata  │              │
│   └─────────┘  └──────────┘  └──────────┘              │
├─────────────────────────────────────────────────────────┤
│                序列化层                                  │
│              Protocol Buffers                           │
│      ┌──────────┐    ┌──────────┐                      │
│      │ 编码/解码 │    │ 压缩/解压 │                      │
│      │Marshal   │    │ gzip     │                      │
│      └──────────┘    └──────────┘                      │
├─────────────────────────────────────────────────────────┤
│                传输层                                    │
│              HTTP/2                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│   │ 多路复用  │  │ 流控     │  │ 头部压缩  │            │
│   │Multiplex │  │Flow Ctrl │  │ HPACK    │            │
│   └──────────┘  └──────────┘  └──────────┘            │
│   ┌──────────┐  ┌──────────┐                           │
│   │ 帧协议   │  │ TLS/ALPN │                           │
│   │ Framing  │  │ 安全传输  │                           │
│   └──────────┘  └──────────┘                           │
├─────────────────────────────────────────────────────────┤
│                TCP/IP 网络层                             │
└─────────────────────────────────────────────────────────┘
```

### 一行 gRPC 调用的完整旅程

```go
// 你的业务代码，就这一行
resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "world"})
```

这行代码背后的旅程：

```
客户端                                            服务端
  │                                                 │
  │  1. 拦截器链执行（认证/日志/限流）                  │
  │  2. Protobuf 序列化 HelloRequest                 │
  │  3. Resolver 解析目标地址                          │
  │  4. Balancer/Picker 选择后端                      │
  │  5. 获取/创建 HTTP/2 连接                         │
  │  6. 构造 gRPC 帧（Length-Prefixed-Message）       │
  │  7. 通过 HTTP/2 Stream 发送                      │
  │ ─────────── HEADERS + DATA 帧 ─────────────►     │
  │                                                  │  8. HTTP/2 解帧
  │                                                  │  9. gRPC 解包
  │                                                  │  10. Protobuf 反序列化
  │                                                  │  11. 服务端拦截器链
  │                                                  │  12. 执行业务逻辑
  │                                                  │  13. 原路返回
  │ ◄────── HEADERS + DATA + TRAILERS 帧 ─────────── │
  │  14. 解帧、解码、返回结果                           │
  ▼                                                  ▼
```

---

## 1.4 grpc-go 项目结构总览与源码导读地图

### 目录结构全景

```
grpc-go/
├── clientconn.go          # 客户端核心：ClientConn、NewClient、addrConn
├── server.go              # 服务端核心：Server、Serve、RegisterService
├── call.go                # 一元调用入口：Invoke → invoke
├── stream.go              # 流调用：ClientStream、ServerStream、NewStream
├── interceptor.go         # 拦截器类型定义
├── dialoptions.go         # 拨号选项（函数式选项模式）
├── rpc_util.go            # CallOption、压缩、消息大小限制
├── codec.go               # 编解码桥接
├── picker_wrapper.go      # Picker 包装器（负载均衡选择）
├── resolver_wrapper.go    # Resolver → ClientConn 桥接
├── balancer_wrapper.go    # Balancer → ClientConn 桥接
├── service_config.go      # 服务配置解析
│
├── internal/
│   └── transport/         # ★ HTTP/2 传输层（核心中的核心）
│       ├── transport.go       # ClientTransport/ServerTransport 接口、Stream 结构
│       ├── http2_client.go    # 客户端 HTTP/2 传输实现
│       ├── http2_server.go    # 服务端 HTTP/2 传输实现
│       ├── controlbuf.go      # loopyWriter（写调度器）
│       ├── flowcontrol.go     # 流量控制实现
│       ├── bdp_estimator.go   # 带宽延迟积估算器
│       └── handler_server.go  # HTTP Handler 模式传输
│
├── resolver/              # 命名解析
│   ├── resolver.go            # Builder/Resolver/ClientConn 接口
│   ├── dns/                   # DNS 解析器
│   ├── passthrough/           # 直通解析器
│   └── manual/                # 手动解析器（测试用）
│
├── balancer/              # 负载均衡
│   ├── balancer.go            # Builder/Balancer/Picker/SubConn 接口
│   ├── base/                  # 基础框架
│   ├── roundrobin/            # 轮询
│   ├── pickfirst/             # 选首个
│   ├── grpclb/               # gRPC LB 协议
│   └── ringhash/             # 一致性哈希
│
├── credentials/           # 安全认证
│   ├── credentials.go         # TransportCredentials/PerRPCCredentials 接口
│   ├── tls.go                 # TLS 凭证
│   ├── oauth/                 # OAuth2
│   └── insecure/              # 不安全（明文）
│
├── encoding/              # 编码/压缩
│   ├── encoding.go            # Codec/Compressor 注册表
│   ├── proto/                 # Protobuf 编解码器
│   └── gzip/                  # Gzip 压缩器
│
├── codes/                 # 状态码定义（0-16）
├── status/                # Status 错误类型
├── metadata/              # 元数据（HTTP/2 头部）
├── connectivity/          # 连接状态枚举
├── health/                # 健康检查协议
├── keepalive/             # 保活机制
├── xds/                   # xDS 动态配置
└── examples/              # ★ 官方示例（学习起点）
    ├── helloworld/            # 最简一元调用
    └── route_guide/           # 四种调用模式完整示例
```

### 关键源码导读路径

理解 gRPC 内部原理，需要掌握以下几条核心代码路径：

**路径1：一元 RPC 调用链（最简路径）**
```
client.SayHello()                          # 生成的 Stub 代码
  → ClientConn.Invoke()                    # call.go:29
    → interceptor / invoke()               # call.go:35-37
      → newClientStream()                  # stream.go
        → Picker.pick() 选择 SubConn       # picker_wrapper.go
          → transport.NewStream()          # internal/transport/http2_client.go
            → loopyWriter 写 HEADERS+DATA  # internal/transport/controlbuf.go
              → HTTP/2 帧发送到网络
```

**路径2：服务端处理链**
```
HTTP/2 连接到达
  → http2Server.operateHeaders()           # internal/transport/http2_server.go
    → Server.handleStream()                # server.go
      → 服务路由匹配
        → 拦截器链
          → 业务 Handler
            → 原路返回 Response
```

**路径3：连接建立与状态管理**
```
NewClient(target)
  → initParsedTargetAndResolverBuilder()   # clientconn.go
    → exitIdleMode()
      → ccResolverWrapper.start()          # resolver_wrapper.go
        → Resolver.UpdateState()           # 解析出地址列表
          → ccBalancerWrapper.updateClientConnState()  # balancer_wrapper.go
            → Balancer 创建 SubConn
              → addrConn.connect()         # 建立物理连接
                → transport.newHTTP2Client()  # HTTP/2 握手
```

---

## 1.5 动手实验：编译运行与抓包观察

### 实验1：运行 HelloWorld 示例

```bash
# 进入 helloworld 示例目录
cd examples/helloworld

# 查看项目结构
ls -la greeter_client/
ls -la greeter_server/

# 启动服务端（在一个终端）
go run greeter_server/main.go

# 启动客户端（在另一个终端）
go run greeter_client/main.go
```

预期输出：
```
# 服务端
2024/01/01 10:00:00 server listening at [::]:50051
2024/01/01 10:00:01 Received: world

# 客户端
2024/01/01 10:00:01 Greeting: Hello world
```

### 实验2：用环境变量开启 gRPC 调试日志

gRPC 内置了详细的调试日志系统，通过环境变量开启：

```bash
# 开启所有 gRPC 日志
export GRPC_GO_LOG_VERBOSITY_LEVEL=99
export GRPC_GO_LOG_SEVERITY_LEVEL=info

# 重新运行客户端
go run greeter_client/main.go
```

你会看到类似输出：
```
INFO: [core] parsed dial target is: {...}
INFO: [core] Channel authority set to "localhost:50051"
INFO: [core] ccResolverWrapper: sending update to ClientConn
INFO: [core] pickfirstBalancer: HandleSubConnStateChange: 0xc000..., CONNECTING
INFO: [core] pickfirstBalancer: HandleSubConnStateChange: 0xc000..., READY
INFO: [core] Subchannel Connectivity change to READY
```

**关键观察点**：
1. `parsed dial target` — 目标解析过程
2. `pickfirstBalancer` — 默认的负载均衡策略
3. `CONNECTING → READY` — 连接状态转换

### 实验3：使用 channelz 观察 Channel 内部状态

channelz 是 gRPC 内置的连接自省工具：

```go
// 在客户端代码中添加 channelz 服务
import "google.golang.org/grpc/channelz/service"

func main() {
    // ... 创建 ClientConn 之后 ...
    lis, _ := net.Listen("tcp", ":50052")
    czServer := grpc.NewServer()
    service.RegisterChannelzService(czServer)
    go czServer.Serve(lis)

    // 浏览器访问 http://localhost:50052
    // 可以看到 Channel、SubChannel、Socket 的详细状态
}
```

### 实验4：用 Wireshark 抓包观察 HTTP/2 帧

1. 安装 Wireshark，选择 Loopback 接口（本地通信）
2. 过滤条件：`tcp.port == 50051`
3. 右键 → Decode As → HTTP/2
4. 运行 HelloWorld 示例，观察：

**你将看到的关键帧**：

```
1. [TCP SYN/SYN-ACK/ACK]          # TCP 三次握手
2. [TLS ClientHello/ServerHello]   # TLS 握手（如果启用）
3. [HTTP/2 SETTINGS]               # HTTP/2 初始设置帧
4. [HTTP/2 HEADERS]                # gRPC 请求头
   :method = POST
   :path = /helloworld.Greeter/SayHello
   :authority = localhost:50051
   content-type = application/grpc
   te = trailers
   grpc-encoding = identity
5. [HTTP/2 DATA]                   # gRPC 请求体（Protobuf 编码）
6. [HTTP/2 HEADERS]                # gRPC 响应头
7. [HTTP/2 DATA]                   # gRPC 响应体
8. [HTTP/2 HEADERS (trailers)]     # gRPC 尾部状态
   grpc-status = 0                 # 0 = OK
```

> **注意**：如果不使用 TLS，gRPC 使用 h2c（HTTP/2 明文），Wireshark 可以直接解析。
> 如果使用 TLS，需要配置密钥日志文件让 Wireshark 解密。

### 实验5：在代码中添加 channelz 状态监听

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/connectivity"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    // 创建连接
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithDefaultCallOptions(),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    // 观察连接状态变化
    go func() {
        for {
            state := conn.GetState()
            fmt.Printf("Channel State: %s\n", state)
            conn.WaitForStateChange(context.Background(), state)
        }
    }()

    // 发起调用
    client := pb.NewGreeterClient(conn)
    resp, err := client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Printf("Greeting: %s\n", resp.GetMessage())

    time.Sleep(time.Second)
}
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| RPC | 远程调用的本质：序列化 + 传输 + 寻址 + 错误处理 |
| gRPC 设计 | 服务优先、HTTP/2 原生、语言中立、面向失败 |
| 技术栈 | Protobuf（序列化）+ HTTP/2（传输）+ 帧协议 |
| 源码结构 | clientconn.go/server.go 为入口，internal/transport/ 为核心 |
| 调用链路 | Invoke → newClientStream → Pick → Transport → HTTP/2 Frame |

---

## 下节预告

第2课将深入 **Protocol Buffers 编码原理**，理解 gRPC 消息在网络上究竟长什么样——从 Varint 到 TLV，手写二进制解码。
