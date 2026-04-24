# 第4课：Channel 与连接管理

## 学习目标

- 掌握 ClientConn 的创建与状态机转换
- 理解 Resolver 如何将目标 URI 解析为地址列表
- 追踪连接建立的完整过程
- 理解连接复用与空闲管理

---

## 4.1 ClientConn 的创建与状态机

### ClientConn 是什么？

`ClientConn` 是 gRPC 客户端的"虚拟通道"——它不代表单个网络连接，而是管理一组连接的抽象层。

```go
// clientconn.go 中 ClientConn 的核心字段
type ClientConn struct {
    ctx    context.Context
    cancel context.CancelFunc

    target    string                // 目标 URI（如 "dns:///example.com:443"）
    parsedTarget resolver.Target    // 解析后的目标
    authority string               // HTTP/2 :authority 值

    dopts    dialOptions            // 拨号选项
    conns    map[*addrConn]struct{} // 活跃的物理连接集合

    resolverWrapper *ccResolverWrapper  // 命名解析器包装
    balancerWrapper *ccBalancerWrapper  // 负载均衡器包装
    pickerWrapper   *pickerWrapper      // Picker 包装

    csMgr    *connectivityStateManager  // 连接状态管理器
    idlenessMgr *idle.Manager           // 空闲管理器

    retryThrottler atomic.Pointer[retryThrottler]  // 重试限流器
}
```

### NewClient() 创建过程

```go
// clientconn.go:185 — NewClient 的核心流程
func NewClient(target string, opts ...DialOption) (*ClientConn, error) {
    cc := &ClientConn{
        target: target,
        conns:  make(map[*addrConn]struct{}),
        dopts:  defaultDialOptions(),
    }

    // 1. 应用全局和用户拨号选项
    for _, opt := range opts {
        opt.apply(&cc.dopts)
    }

    // 2. 解析目标 URI，确定 Resolver Builder
    cc.initParsedTargetAndResolverBuilder()

    // 3. 组装拦截器链
    chainUnaryClientInterceptors(cc)
    chainStreamClientInterceptors(cc)

    // 4. 验证传输凭证
    cc.validateTransportCredentials()

    // 5. 初始化空闲状态（不立即连接！）
    cc.initIdleStateLocked()
    cc.idlenessMgr = idle.NewManager((*idler)(cc), cc.dopts.idleTimeout)

    return cc, nil
}
```

**关键设计**：`NewClient()` 是**惰性的**——它不会立即建立任何网络连接！连接在第一次 RPC 调用时按需建立。

### Channel / SubConn / TCP 的层次关系

很多初学者会以为 Channel 对应一条 TCP 连接，但实际上 gRPC 采用的是三层架构：

```
ClientConn (Channel — 虚拟连接)
  ├── SubConn (acBalancerWrapper — LB 的最小单元)
  │     └── addrConn — "a network connection to a given address"
  │           └── transport ← 真正的 TCP/HTTP2 连接
  ├── SubConn
  │     └── addrConn
  │           └── transport
  └── SubConn
        └── addrConn
              └── transport
```

| 层级 | 数量 | 对应关系 |
|------|------|---------|
| Channel (ClientConn) | 1 | 一次 `Dial` 创建一个 |
| SubConn | N | 每个后端实例一个 |
| TCP 连接 (transport) | N | 每个 SubConn 内部一个 |

**为什么一个 Channel 能有多条 TCP 连接？**

考虑这个场景：
```
你的服务 example-svc 部署了 3 个 Pod:
  10.0.1.1:50051
  10.0.1.2:50051
  10.0.1.3:50051
```

当 `grpc.Dial("example-svc:50051")` 时：
1. **Resolver** 把服务名解析为 3 个地址
2. **Balancer** 为每个地址创建一个 SubConn → addrConn → 各自建立一条 TCP 连接
3. 每次 RPC 时，**Picker** 从 3 个 Ready 的 SubConn 中选一个发请求

这就是客户端负载均衡的核心机制。Channel 其实更像一个**连接池管理器**。

**SubConn 的精确定义**

```go
// balancer/balancer.go:140
// ClientConn.NewSubConn is called by balancer to create a new SubConn.

// balancer/balancer.go:280
// SubConn is the connection to use for this pick, if its state is Ready.
```

SubConn 是 Load Balancer 创建的子连接，包裹 `addrConn`。它是负载均衡的**最小单元**——每个 SubConn 对应一个后端地址，维护一条 transport（TCP 连接），并拥有独立的连接状态。

### 连接状态机

`connectivity/connectivity.go` 定义了五个状态：

```
                    ┌──────────┐
          首次RPC    │   IDLE   │ ◄── 新建 ClientConn 的初始状态
          或Connect  └────┬─────┘
                       │
                       ▼
                 ┌──────────┐
                 │CONNECTING│ ◄── 解析目标地址、建立连接
                 └────┬─────┘
                      │
              ┌───────┴───────┐
              │               │
              ▼               ▼
        ┌─────────┐    ┌────────────────┐
        │  READY   │    │TRANSIENTFAILURE│ ◄── 连接失败
        │(可发RPC) │    │ (临时失败)      │
        └────┬────┘    └───────┬────────┘
             │                 │
             │  连接断开        │  重试成功
             ▼                 │
        ┌──────────┐           │
        │CONNECTING│◄──────────┘
        └────┬─────┘
             │
             ▼ Close()
        ┌──────────┐
        │ SHUTDOWN │ ◄── ClientConn.Close() 后的终态
        └──────────┘
```

**状态转换规则**（源码：`clientconn.go` 中的 `exitIdleMode()` 和 `enterIdleMode()`）：
- `IDLE → CONNECTING`：首次 RPC 或调用 `Connect()` 触发 `exitIdleMode()`
- `CONNECTING → READY`：至少一个 SubConn 就绪
- `CONNECTING → TRANSIENT_FAILURE`：所有 SubConn 连接失败
- `READY → IDLE`：所有 RPC 完成后空闲超时
- `TRANSIENT_FAILURE → CONNECTING`：重连中
- `任何状态 → SHUTDOWN`：调用 `Close()`

---

## 4.2 命名解析（Resolver）

### Resolver 的角色

Resolver 负责将目标 URI 转换为一组后端地址：

```
"dns:///example.com:443"
        │
        ▼  DNS Resolver
[10.0.0.1:443, 10.0.0.2:443, 10.0.0.3:443]
```

### Resolver 接口

`resolver/resolver.go` 定义了核心接口：

```go
// Builder 创建 Resolver 实例
type Builder interface {
    Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
    Scheme() string  // 如 "dns", "passthrough", "manual"
}

// Resolver 执行实际的解析工作
type Resolver interface {
    ResolveNow(ResolveNowOptions)  // 立即重新解析
    Close()                         // 关闭
}

// ClientConn 是 Resolver 回写结果的接口
type ClientConn interface {
    UpdateState(State) error  // ★ 解析出新地址后调用
    NewAddress(addrs []Address)
    // ...
}

// State 包含解析结果
type State struct {
    Addresses    []Address         // 后端地址列表
    ServiceConfig *serviceconfig.ParseResult  // 服务配置
    Attributes   *attributes.Attributes       // 附加属性
}
```

### 内置 Resolver

```
Resolver           Scheme        行为
─────────────────────────────────────────────────────────────
dns                dns:///       DNS A/AAAA 记录查询
passthrough        passthrough:///  直接将目标字符串当地址
manual             manual:///    手动控制（测试用）
unix               unix:///      Unix 域套接字
xDS                xds:///       xDS 动态发现
```

### DNS 服务器选择机制

gRPC 的 DNS 解析器**不自建 DNS 协议**，而是委托给 Go 标准库的 `net.Resolver`。DNS 服务器的选择取决于拨号目标中是否指定了 **authority**。

**关键入口** (`dns_resolver.go:147`):

```go
d.resolver, err = internal.NewNetResolver(target.URL.Host)
//                                       ↑ authority 来自 URI 的 host 部分
```

**分支逻辑** (`dns_resolver.go:92`):

```go
var newNetResolver = func(authority string) (internal.NetResolver, error) {
    if authority == "" {
        return net.DefaultResolver, nil   // 分支 A: 系统默认 DNS
    }

    host, port, _ := parseTarget(authority, "53")
    authorityWithPort := net.JoinHostPort(host, port)

    return &net.Resolver{
        PreferGo: true,                                      // 分支 B: Go 纯客户端
        Dial:     internal.AddressDialer(authorityWithPort), // 指定 DNS 服务器
    }, nil
}
```

**场景 A — 不指定 authority（绝大多数情况）**

```
grpc.Dial("example.com:50051")
```
→ `target.URL.Host` 为空 → 使用 `net.DefaultResolver`

DNS 服务器由操作系统决定：
- Linux：`/etc/resolv.conf` 中的 `nameserver`
- Windows：网络适配器的 DNS 设置
- Go 的 `DefaultResolver` 优先级：CGO enabled 时走 libc getaddrinfo()，CGO disabled 时走纯 Go DNS 客户端直接读 `/etc/resolv.conf`

**场景 B — 指定 authority（自定义 DNS 服务器）**

```
grpc.Dial("dns://8.8.8.8/example.com:50051")
grpc.Dial("dns://dns.local:5353/example.com:50051")
```
→ `target.URL.Host` = `"8.8.8.8"` → 创建自定义 `net.Resolver`，`PreferGo: true`，直接向指定地址发起 DNS 查询

| 拨号方式 | DNS 服务器 | 解析器 |
|---------|-----------|--------|
| `example.com:50051` | OS 默认（/etc/resolv.conf） | `net.DefaultResolver` |
| `dns://8.8.8.8/example.com:50051` | `8.8.8.8:53` | 自定义 `net.Resolver` |
| `dns://ns1.local:5353/example.com:50051` | `ns1.local:5353` | 自定义 `net.Resolver` |

### Resolver → ClientConn 的桥接

`resolver_wrapper.go` 中的 `ccResolverWrapper` 桥接 Resolver 和 ClientConn：

```go
// resolver_wrapper.go
type ccResolverWrapper struct {
    cc       *ClientConn                      // 指向 ClientConn
    resolver resolver.Resolver                 // 实际的 Resolver
    curState resolver.State                    // 当前解析状态
    serializer *grpcsync.CallbackSerializer    // 序列化回调
}

// start() 启动 Resolver
func (ccr *ccResolverWrapper) start() error {
    // 根据 scheme 构建 Resolver
    ccr.resolver, err = ccr.cc.resolverBuilder.Build(
        ccr.cc.parsedTarget, ccr, opts)
    return err
}

// UpdateState() — Resolver 发现新地址后调用
func (ccr *ccResolverWrapper) UpdateState(s resolver.State) error {
    ccr.mu.Lock()
    ccr.curState = s
    ccr.mu.Unlock()

    // 将解析结果传播到 ClientConn
    ccr.cc.updateResolverStateAndUnlock(s, nil)
    return nil
}
```

### 目标 URI 解析过程

```
NewClient("dns:///example.com:443")
  │
  ├── 解析 URI: scheme="dns", endpoint="example.com:443"
  │
  ├── 查找 Resolver Builder: resolver.Get("dns") → dns.resolverBuilder
  │
  └── exitIdleMode() → resolverWrapper.start()
        │
        └── dns.Builder.Build() → 启动 DNS 查询
              │
              ├── 查询 DNS A 记录: example.com → 10.0.0.1, 10.0.0.2
              │
              └── UpdateState({Addresses: [{Addr: "10.0.0.1:443"}, {Addr: "10.0.0.2:443"}]})
                    │
                    └── ClientConn.updateResolverStateAndUnlock()
                          │
                          └── balancerWrapper.updateClientConnState()
                                │
                                └── Balancer 创建/更新 SubConn
```

---

## 4.3 连接建立过程

### addrConn：物理连接的表示

`addrConn` 是 `ClientConn` 内部管理单个后端地址连接的结构：

```go
// clientconn.go（简化版）
type addrConn struct {
    cc     *ClientConn          // 所属的 ClientConn
    addrs  []resolver.Address   // 关联的地址列表

    transport transport.ClientTransport  // HTTP/2 传输层

    state   connectivity.State   // 当前连接状态
    backoff backoff.Strategy     // 退避策略
}
```

### 连接建立链路

```
addrConn.connect()
  │
  ├── 创建 HTTP/2 传输
  │   └── transport.newHTTP2Client()
  │         │
  │         ├── net.Dial() 建立 TCP 连接
  │         │
  │         ├── TLS 握手（如果启用）
  │         │   └── credentials.ClientHandshake()
  │         │
  │         ├── 发送 HTTP/2 连接前言
  │         │   └── "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
  │         │
  │         ├── 发送 SETTINGS 帧
  │         │   ├── SETTINGS_MAX_CONCURRENT_STREAMS = 100
  │         │   ├── SETTINGS_INITIAL_WINDOW_SIZE = 65535
  │         │   └── SETTINGS_MAX_FRAME_SIZE = 16384
  │         │
  │         ├── 接收服务端 SETTINGS
  │         │
  │         ├── 发送 SETTINGS ACK
  │         │
  │         └── 启动读取协程 reader()
  │             └── 持续读取 HTTP/2 帧
  │
  └── 连接就绪
      └── 更新 SubConn 状态为 READY
          └── 触发 Balancer 更新 Picker
```

### 源码追踪：newHTTP2Client

`internal/transport/http2_client.go` 中 `newHTTP2Client()` 的核心逻辑：

```go
func newHTTP2Client(connectCtx, ctx context.Context, ...) (*http2Client, error) {
    // 1. 建立 TCP 连接（含代理支持）
    conn, err := dial(connectCtx, opts.Dialer, opts.BaseConfig.Address, ...)
    // 如果使用代理，先发 HTTP CONNECT

    // 2. TLS 握手
    if creds := opts.TransportCredentials; creds != nil {
        conn, authInfo, err = creds.ClientHandshake(connectCtx,
            opts.BaseConfig.Authority, conn)
    }

    // 3. 初始化 HTTP/2 帧读取器
    framer := newFramer(conn, writeBufSize, readBufSize, ...)

    // 4. 发送 HTTP/2 连接前言 + SETTINGS
    clientPreface := []byte(http2.ClientPreface)
    conn.Write(clientPreface)
    framer.writeSettings(...)  // 发送初始 SETTINGS

    // 5. 启动读循环
    go t.reader()

    // 6. 启动 keepalive pinger
    if kp.Time != infinity {
        go t.keepalive()
    }

    return t, nil
}
```

---

## 4.4 连接复用与多 Stream 并发

### 一个连接上的多路复用

HTTP/2 的核心优势：一个 TCP 连接上可以并发多个 Stream（RPC）：

```
TCP 连接
  ├── Stream 1: SayHello RPC    (客户端发起, ID=1)
  ├── Stream 3: SayHello RPC    (客户端发起, ID=3)
  ├── Stream 5: ListFeatures RPC(客户端发起, ID=5)
  └── ... 直到 maxConcurrentStreams

每个 Stream 的帧交错传输：
  连接上: [HEADERS(1)][DATA(1)][HEADERS(3)][DATA(5)][DATA(1)][HEADERS(5)]...
  通过 Stream ID 解复用
```

### Stream ID 的管理

```go
// internal/transport/http2_client.go
func (t *http2Client) getStream(id uint32) *Stream {
    // Stream ID 是奇数，客户端发起
    // 1, 3, 5, 7, ...
    t.mu.Lock()
    s := t.activeStreams[id]
    t.mu.Unlock()
    return s
}

// 下一个 Stream ID
func (t *http2Client) nextID() uint32 {
    t.nextID += 2  // 偶数 ID 保留给服务端（PUSH_PROMISE，gRPC 不使用）
    return t.nextID
}
```

### 默认连接参数

`internal/transport/defaults.go`：

```go
const (
    defaultWindowSize           = 65535   // HTTP/2 默认流控窗口
    defaultClientMaxReceiveMessageSize = 4 * 1024 * 1024  // 4MB
    defaultMaxStreamsClient     = 100     // 最大并发流数
    defaultWriteQuota           = 64 * 1024  // 64KB 写配额
    defaultClientMaxHeaderListSize = 16 << 20  // 16MB
)
```

### 空闲连接管理

ClientConn 内置空闲管理——长时间无 RPC 时自动关闭连接以节省资源：

```go
// clientconn.go
func (cc *ClientConn) enterIdleMode() {
    // 1. 关闭 Resolver
    rWrapper.close()
    // 2. 重置 Picker
    cc.pickerWrapper.reset()
    // 3. 关闭 Balancer
    bWrapper.close()
    // 4. 更新状态为 IDLE
    cc.csMgr.updateState(connectivity.Idle)
    // 5. 重建内部状态
    cc.initIdleStateLocked()
    // 6. 关闭所有 SubConn（物理连接）
    for ac := range conns {
        ac.tearDown(errConnIdling)
    }
}
```

空闲超时可通过 `WithIdleTimeout` DialOption 配置：

```go
conn, err := grpc.NewClient(target,
    grpc.WithIdleTimeout(5*time.Minute), // 5分钟无 RPC 则进入空闲
)
```

---

## 4.5 动手实验：编写 Resolver 插件

### 实验：实现一个静态文件 Resolver

```go
package resolver

import (
    "context"
    "encoding/json"
    "fmt"
    "os"

    "google.golang.org/grpc/resolver"
)

const fileScheme = "file"

// fileBuilder 从 JSON 文件读取后端地址列表
type fileBuilder struct{}

func (b *fileBuilder) Build(target resolver.Target,
    cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {

    r := &fileResolver{
        target: target,
        cc:     cc,
    }

    // 立即执行一次解析
    r.resolve()

    return r, nil
}

func (b *fileBuilder) Scheme() string { return fileScheme }

type fileResolver struct {
    target resolver.Target
    cc     resolver.ClientConn
}

func (r *fileResolver) resolve() {
    // 从文件读取地址列表
    // 文件格式: ["10.0.0.1:50051", "10.0.0.2:50051"]
    filename := r.target.URL.Opaque // file:///path/to/addresses.json
    data, err := os.ReadFile(filename)
    if err != nil {
        r.cc.ReportError(err)
        return
    }

    var addrs []string
    json.Unmarshal(data, &addrs)

    var resolvedAddrs []resolver.Address
    for _, addr := range addrs {
        resolvedAddrs = append(resolvedAddrs,
            resolver.Address{Addr: addr})
    }

    // ★ 将解析结果报告给 ClientConn
    r.cc.UpdateState(resolver.State{
        Addresses: resolvedAddrs,
    })
}

func (r *fileResolver) ResolveNow(opts resolver.ResolveNowOptions) {
    r.resolve() // 重新读取文件
}

func (r *fileResolver) Close() {}

func init() {
    resolver.Register(&fileBuilder{})
}
```

使用方式：

```go
package main

import (
    "log"
    "google.golang.org/grpc"
    _ "yourpackage/resolver"  // 注册 file resolver
)

func main() {
    // 目标格式: scheme:///path
    conn, err := grpc.NewClient("file:///etc/grpc/backends.json")
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()

    // 当 backends.json 变更时，调用 ResolveNow 触发重新解析
    // conn.ResolveNow(resolver.ResolveNowOptions{})
}
```

### 实验：观察连接状态变化

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

func watchState(conn *grpc.ClientConn) {
    for {
        state := conn.GetState()
        fmt.Printf("[%s] Channel state: %s\n",
            time.Now().Format("15:04:05.000"), state)

        // 阻塞等待状态变化
        conn.WaitForStateChange(context.Background(), state)
    }
}

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithIdleTimeout(30*time.Second), // 缩短空闲超时方便观察
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    go watchState(conn)

    // 发起 RPC —— 触发 IDLE → CONNECTING → READY
    client := pb.NewGreeterClient(conn)
    resp, err := client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Printf("Greeting: %s\n", resp.GetMessage())

    // 等待空闲超时 —— 观察 READY → IDLE
    fmt.Println("Waiting for idle timeout...")
    time.Sleep(60 * time.Second)

    // 再次 RPC —— 观察 IDLE → CONNECTING → READY
    resp, err = client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "again"})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Printf("Greeting: %s\n", resp.GetMessage())
}
```

预期输出：
```
[10:00:00.000] Channel state: IDLE
[10:00:00.050] Channel state: CONNECTING
[10:00:00.100] Channel state: READY
Greeting: Hello world
Waiting for idle timeout...
[10:00:30.100] Channel state: IDLE          ← 空闲超时
[10:01:30.050] Channel state: CONNECTING     ← 再次 RPC
[10:01:30.100] Channel state: READY
Greeting: Hello again
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| ClientConn | 虚拟通道，管理一组连接，惰性创建 |
| 状态机 | IDLE → CONNECTING → READY/TRANSIENT_FAILURE → SHUTDOWN |
| Resolver | 将 URI 解析为 []Address，通过 UpdateState 报告 |
| ccResolverWrapper | 桥接 Resolver 和 ClientConn，序列化回调 |
| 连接建立 | Dial → TLS → HTTP/2 Preface → SETTINGS → Ready |
| 空闲管理 | 无 RPC 时自动关闭连接，有新 RPC 时重建 |

---

## 下节预告

第5课将深入 **负载均衡与服务发现**，理解 Resolver 的地址列表如何传递给 Balancer，Picker 如何选择后端，以及 xDS 动态负载均衡。
