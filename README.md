# gRPC 深入原理 — 十节课课程大纲

## 课程文件索引

| 课次 | 主题 | 文件 |
|------|------|------|
| 第1课 | gRPC 全景概览与 RPC 基础 | [lesson01-overview.md](lesson01-overview.md) |
| 第2课 | Protocol Buffers 深度解析 | [lesson02-protobuf.md](lesson02-protobuf.md) |
| 第3课 | HTTP/2 与 gRPC 帧协议 | [lesson03-http2.md](lesson03-http2.md) |
| 第4课 | Channel 与连接管理 | [lesson04-channel.md](lesson04-channel.md) |
| 第5课 | 负载均衡与服务发现 | [lesson05-loadbalancing.md](lesson05-loadbalancing.md) |
| 第6课 | 四种调用模式与流控机制 | [lesson06-streaming.md](lesson06-streaming.md) |
| 第7课 | 拦截器与中间件模式 | [lesson07-interceptor.md](lesson07-interceptor.md) |
| 第8课 | 安全机制：TLS、Token 与认证 | [lesson08-security.md](lesson08-security.md) |
| 第9课 | 可靠性机制：重试、超时与死线传播 | [lesson09-reliability.md](lesson09-reliability.md) |
| 第10课 | 高级架构 — xDS、多语言与生产实践 | [lesson10-advanced.md](lesson10-advanced.md) |

---

## 课程大纲

## 第1课：gRPC 全景概览与 RPC 基础
- RPC 的演进：从本地调用到远程调用
- gRPC 的诞生背景与设计哲学（与 REST/GraphQL 对比）
- gRPC 核心技术栈：HTTP/2 + Protobuf + 帧协议
- grpc-go 项目结构总览与源码导读地图
- 动手：编译运行 grpc-go 示例，用 Wireshark 抓包观察原始帧

## 第2课：Protocol Buffers 深度解析
- .proto 文件到 Go 代码的完整编译链路
- 编码原理：Varint、Tag-Length-Value、字段布局
- 前后兼容性设计：required/optional/repeated 的演化语义
- 生成的 Stub 代码结构解读（Client/Server 接口）
- 动手：手写二进制解码，验证 Protobuf 编码格式

## 第3课：HTTP/2 与 gRPC 帧协议
- HTTP/2 核心概念：Stream、Frame、Flow Control
- gRPC 如何映射到 HTTP/2：请求头、消息帧、尾部状态
- gRPC 帧格式：Length-Prefixed-Message（压缩标志 + 消息长度 + 消息体）
- Trailers 机制与 gRPC Status 传递
- 动手：用 h2c 或 Go 的 golang.org/x/net/http2 直接构造 gRPC 请求

## 第4课：Channel 与连接管理
- ClientConn 的创建与状态机（Idle → Connecting → Ready → TransientFailure → Shutdown）
- 命名解析（Resolver）：从 URI 到地址列表（DNS/手动解析器）
- 连接建立过程：拨号 → HTTP/2 握手 → Settings 协商
- 连接复用与多 Stream 并发
- 源码追踪：ClientConn → addrConn → transport.newHTTP2Client
- 动手：编写 Resolver 插件，观察连接状态变化

## 第5课：负载均衡与服务发现
- 负载均衡架构：Client-side LB vs Proxy-side LB
- Balancer 接口与实现：RoundRobin、PickFirst
- SubConn 状态管理与 Picker 机制
- 负载均衡策略切换：grpclb、xDS（后端流量管理）
- 服务发现与负载均衡的协作：Resolver → Balancer → Picker
- 源码追踪：ccResolverWrapper → ccBalancerWrapper → picker
- 动手：实现自定义 Balancer，观察请求分布

## 第6课：四种调用模式与流控机制
- Unary RPC：最简调用路径全链路追踪
- Server Streaming：服务端推送与读取循环
- Client Streaming：客户端批量发送与半关闭
- Bidirectional Streaming：全双工通信与并发读写
- 流的背压与 Flow Control：HTTP/2 窗口更新帧
- 源码追踪：ClientStream / ServerStream → transport.Stream → loopyWriter
- 动手：实现双向流聊天，用 Wireshark 观察 Flow Control 窗口变化

## 第7课：拦截器（Interceptor）与中间件模式
- 一元拦截器 vs 流拦截器（Unary/Stream Interceptor）
- 拦截器链的执行模型与调用顺序
- 常见拦截器实现：认证、日志、限流、指标采集
- 拦截器与 gRPC 内部机制的交互（如重试、重定向）
- 源码追踪：invoke → chainUnaryClientInterceptors → 最终 RPC 调用
- 动手：实现链式拦截器，含 Auth + Logging + Recovery

## 第8课：安全机制：TLS、Token 与认证
- TLS 握手与 ALPN 协商 HTTP/2
- 证书体系：单向认证 vs 双向认证（mTLS）
- Token 认证：Per-RPC Credentials（OAuth2/JWT）
- CallCredentials 与 ChannelCredentials 的组合
- 源码追踪：credentials 包 → TransportCredentials → Peer 信息注入
- 动手：配置 mTLS 双向认证，用自签证书部署安全 gRPC 服务

## 第9课：可靠性机制：重试、超时与死线传播
- Deadline 与超时传播：grpc-timeout 头的逐跳递减
- 重试策略：Retry vs Hedging（对冲）与幂等性要求
- 透明重试 vs 应用层重试的区别
- 电路断路与 Backoff 策略
- 取消传播：客户端取消如何通知服务端（RST_STREAM / CANCEL）
- 源码追踪：retryThrottler → tryRPC → recvMsg
- 动手：模拟网络故障，观察重试行为与超时传播

## 第10课：高级架构 — xDS、多语言与生产实践
- xDS 协议体系：LDS/RDS/CDS/EDS 与控制面交互
- gRPC xDS 实现：服务发现 → 负载均衡 → 路由决策
- 多语言 gRPC 实现对比：grpc-go vs grpc-java vs grpc-core（C 核心）
- 性能调优：连接池、序列化优化、Buffer 复用（sync.Pool）
- 生产部署：健康检查、优雅关闭、监控指标（OpenCensus/Prometheus）
- 源码追踪：xds 包 → xdsClient → xdsBalancer → xdsResolver
- 动手：搭建 xDS 控制面（Istio），实现动态路由与流量切换

---

## 课程设计原则

| 原则 | 说明 |
|------|------|
| **由浅入深** | 从概览→编码→协议→连接→负载→流→拦截→安全→可靠→架构，每课依赖前课知识 |
| **源码驱动** | 每课均含 grpc-go 源码追踪路径，理论对应实际实现 |
| **动手实践** | 每课配有实验，从抓包观察逐步到实现自定义组件 |
| **闭环理解** | 第1课的全景图在后续9课中逐层展开，第10课回归全局架构 |

## 附录

| 附录 | 主题 | 文件 |
|------|------|------|
| 附录A | Stats Handler — 可观测性接口 | [appendix-a-stats-handler.md](appendix-a-stats-handler.md) |
| 附录B | 关于废弃代码的 FAQ | [appendix-b-deprecated-faq.md](appendix-b-deprecated-faq.md) |

---

建议学习节奏：每课 1-2 周，配合 grpc-go 源码阅读，总计约 3-4 个月完成。
