# 第6课：四种调用模式与流控机制

## 学习目标

- 掌握 gRPC 四种调用模式的原理与差异
- 理解 Streaming RPC 底层的 Stream 生命周期
- 掌握 HTTP/2 流量控制与 gRPC 背压机制
- 追踪 SendMsg/RecvMsg 的源码实现

---

## 6.1 四种调用模式概览

### 如何在 .proto 中指定调用模式

调用模式**在 proto 文件中通过 `stream` 关键字定义**，编译后无法在运行时切换。以 `examples/features/proto/echo/echo.proto` 为例：

```protobuf
service Echo {
  // Unary: 两端都没有 stream —— 一问一答
  rpc UnaryEcho(EchoRequest) returns (EchoResponse) {}

  // Server Streaming: 仅响应加 stream —— 一问多答
  rpc ServerStreamingEcho(EchoRequest) returns (stream EchoResponse) {}

  // Client Streaming: 仅请求加 stream —— 多问一答
  rpc ClientStreamingEcho(stream EchoRequest) returns (EchoResponse) {}

  // Bidi Streaming: 两端都加 stream —— 自由收发
  rpc BidirectionalStreamingEcho(stream EchoRequest) returns (stream EchoResponse) {}
}
```

### 生成的 Go 接口对照

| 模式 | proto 写法 | 生成的客户端方法签名 |
|------|-----------|-------------------|
| Unary | `rpc X(A) returns (B)` | `X(ctx, *A) (*B, error)` |
| Server Streaming | `rpc X(A) returns (stream B)` | `X(ctx, *A) (pb.XClient, error)` → 循环 Recv |
| Client Streaming | `rpc X(stream A) returns (B)` | `X(ctx) (pb.XClient, error)` → Send 多次后 CloseAndRecv |
| Bidi | `rpc X(stream A) returns (stream B)` | `X(ctx) (pb.XClient, error)` → 自由 Send/Recv |

### 四种模式简表

```
┌─────────────────┬──────────────┬──────────────┐
│      模式        │   客户端      │   服务端      │
├─────────────────┼──────────────┼──────────────┤
│ Unary           │ 1 个请求      │ 1 个响应      │
│ Server Streaming│ 1 个请求      │ N 个响应      │
│ Client Streaming│ N 个请求      │ 1 个响应      │
│ Bidi Streaming  │ N 个请求      │ N 个响应      │
└─────────────────┴──────────────┴──────────────┘

数据流方向:
  Unary:          C ──req──► S ──resp──► C
  Server Stream:  C ──req──► S ──resp1──resp2──resp3──► C
  Client Stream:  C ──req1──req2──req3──► S ──resp──► C
  Bidi Stream:    C ──req1──req2──►     ◄──resp1──resp2── S
                  C ◄──resp1──resp2──   ──req3──req4──► S
```

### 四种模式的实际适用场景

| 模式 | 典型场景 | 选择理由 |
|------|---------|---------|
| Unary | 查询用户信息、下单、登录 | 最简单，心智负担最低。能用 Unary 就别用流式 |
| Server Streaming | 导出报表、订阅行情、tail 日志、进度通知 | 服务端数据量大，分批推送。客户端边收边处理，不用等全部到达 |
| Client Streaming | 大文件上传、批量数据导入、传感器上报 | 客户端数据量大，分批发送。发完后统一得到结果 |
| Bidi | 聊天、实时协作编辑、游戏同步、音视频通话 | 收发顺序不固定，需要全双工交互 |

**决策口诀**：客户端数据多 → Client Streaming；服务端数据多 → Server Streaming；两边都多/实时 → Bidi；除此之外 → Unary。

### 在 HTTP/2 层面，四种模式都是 Stream

**关键洞察**：gRPC 的四种调用模式在 HTTP/2 层面都是同一个 Stream，区别仅在于：
- 谁先发 DATA 帧
- 发多少个 DATA 帧
- 何时发送 END_STREAM 标志

```
HTTP/2 Stream 视角:

Unary:     [HEADERS][DATA(END_STREAM)] → [HEADERS][DATA][HEADERS(trailers)]
                    ↑ 客户端半关闭                   ↑ 服务端完成

Server:    [HEADERS][DATA(END_STREAM)] → [HEADERS][DATA][DATA]...[HEADERS(trailers)]
                    ↑ 客户端半关闭                   ↑ 多个 DATA 帧

Client:    [HEADERS][DATA][DATA]...[DATA(END_STREAM)] → [HEADERS][DATA][HEADERS(trailers)]
                    ↑ 多个 DATA，最后半关闭                   ↑ 一个响应

Bidi:      [HEADERS] → [HEADERS]
           [DATA] ← → [DATA]     ← 双方可以交替发 DATA
           [DATA] ← → [DATA]
           [DATA(END_STREAM)] →  ← 客户端半关闭
           [DATA][HEADERS(trailers)] ← 服务端完成
```

---

## 6.2 Unary RPC：最简调用路径全链路追踪

### 从 Invoke 到 HTTP/2 帧

```
call.go:29   ClientConn.Invoke()
  │
  ├── 合并 CallOption:  opts = combine(cc.dopts.callOptions, opts)
  │
  ├── 拦截器分支:
  │   if cc.dopts.unaryInt != nil:
  │     cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
  │   else:
  │     invoke(ctx, method, args, reply, cc, opts...)
  │
  ▼
call.go:65   invoke()
  │
  ├── newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
  │   │  StreamDesc{ServerStreams: false, ClientStreams: false}
  │   │
  │   ├── 获取 MethodConfig（超时、重试策略等）
  │   ├── 合并 metadata
  │   ├── Picker 选择 SubConn
  │   ├── transport.NewStream() 创建 HTTP/2 Stream
  │   └── 返回 ClientStream
  │
  ├── cs.SendMsg(req)    ──── 序列化 + 写 DATA 帧
  │   │
  │   ├── proto.Marshal(req)          → Protobuf 编码
  │   ├── 添加 5 字节 Length-Prefixed 头 → [0x00][length][data]
  │   ├── 压缩（如果配置了 gzip）
  │   └── transport.Write()           → loopyWriter 调度发送
  │
  └── cs.RecvMsg(reply)  ──── 读 DATA 帧 + 反序列化
      │
      ├── 读取 5 字节消息头              → [compressed][length]
      ├── 读取 length 字节消息体
      ├── 解压（如果压缩了）
      └── proto.Unmarshal(data, reply) → Protobuf 解码
```

### 核心发现：Unary 也是 Stream！

```go
// call.go:63 — Unary 调用底层也创建了一个 Stream
var unaryStreamDesc = &StreamDesc{ServerStreams: false, ClientStreams: false}

func invoke(ctx context.Context, method string, req, reply any, cc *ClientConn, opts ...CallOption) error {
    cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
    if err != nil {
        return err
    }
    if err := cs.SendMsg(req); err != nil {
        return err
    }
    return cs.RecvMsg(reply)
}
```

**gRPC 的统一抽象**：所有四种模式都通过 `ClientStream` 接口实现。Unary 只是 "只发一个消息、只收一个消息" 的特例。

---

## 6.3 Server Streaming：服务端推送

### .proto 定义

```protobuf
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

### 客户端实现

```go
// 生成的接口
type RouteGuideClient interface {
    ListFeatures(ctx context.Context, in *Rectangle, opts ...grpc.CallOption) (
        RouteGuide_ListFeaturesClient, error)
}

// 使用方式
stream, err := client.ListFeatures(ctx, &pb.Rectangle{...})
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break  // 流结束
    }
    if err != nil {
        log.Fatalf("error: %v", err)
    }
    fmt.Println(feature)
}
```

### HTTP/2 帧交换

```
客户端                                    服务端
  │── HEADERS ─────────────────────────►│
  │   :path: /routeguide.RouteGuide/ListFeatures
  │── DATA (Rectangle, END_STREAM) ───►│  ← 客户端半关闭
  │                                     │
  │◄── HEADERS ────────────────────────│
  │◄── DATA (Feature 1) ───────────────│
  │◄── DATA (Feature 2) ───────────────│
  │◄── DATA (Feature 3) ───────────────│
  │◄── HEADERS (trailers, END_STREAM)──│  ← grpc-status: 0
  │                                     │
  │  stream.Recv() 返回 io.EOF          │
```

---

## 6.4 Client Streaming：客户端批量发送

### .proto 定义

```protobuf
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

### 客户端实现

```go
stream, err := client.RecordRoute(ctx)
for _, point := range points {
    if err := stream.Send(point); err != nil {
        log.Fatalf("send error: %v", err)
    }
}
// 半关闭 + 接收响应
reply, err := stream.CloseAndRecv()
```

### HTTP/2 帧交换

```
客户端                                    服务端
  │── HEADERS ─────────────────────────►│
  │── DATA (Point 1) ──────────────────►│
  │── DATA (Point 2) ──────────────────►│
  │── DATA (Point 3) ──────────────────►│
  │── DATA (END_STREAM) ───────────────►│  ← 客户端半关闭
  │                                     │
  │◄── HEADERS ────────────────────────│
  │◄── DATA (RouteSummary) ────────────│
  │◄── HEADERS (trailers, END_STREAM)──│
```

---

## 6.5 Bidirectional Streaming：全双工通信

### .proto 定义

```protobuf
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

### 客户端实现

```go
stream, err := client.RouteChat(ctx)

// 发送协程
go func() {
    for _, note := range notes {
        stream.Send(note)
    }
    stream.CloseSend()  // 半关闭发送方向
}()

// 接收循环
for {
    note, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("recv error: %v", err)
    }
    fmt.Println(note)
}
```

### 关键：并发读写

`stream.go` 中的 `ClientStream` 接口明确允许：

> It is safe to have a goroutine calling SendMsg and another goroutine calling RecvMsg on the same stream at the same time.

但同一方向的操作（如两个协程同时 SendMsg）是不安全的。

---

## 6.6 流的背压与 Flow Control

### HTTP/2 流量控制机制

HTTP/2 流量控制确保接收方不会被发送方的数据淹没：

```
发送方                               接收方
  │                                    │
  │  初始窗口: 65535 字节               │
  │                                    │
  │── DATA (1000 bytes) ─────────────►│  窗口: 65535 → 64535
  │── DATA (1000 bytes) ─────────────►│  窗口: 64535 → 63535
  │── DATA (1000 bytes) ─────────────►│  窗口: 63535 → 62535
  │           ...                      │
  │                                    │  应用层处理完数据后:
  │◄── WINDOW_UPDATE(3000) ───────────│  窗口恢复: +3000
  │                                    │
  │── DATA (继续发送) ────────────────►│
```

### 两层流量控制

HTTP/2 有两层流量控制窗口：

```
1. 连接级窗口: 控制整个 TCP 连接上的未确认数据量
   初始: 65535 字节

2. Stream 级窗口: 控制单个 Stream 上的未确认数据量
   初始: 65535 字节

发送 DATA 帧时，必须同时满足:
  - 连接级窗口 > 0
  - Stream 级窗口 > 0
```

### gRPC 的 Flow Control 源码

`internal/transport/flowcontrol.go`：

```go
// writeQuota — Stream 级写配额
type writeQuota struct {
    quota int32       // 可用配额
    ch    chan struct{} // 配额可用时通知
}

func (w *writeQuota) get(sz int32) error {
    for {
        if atomic.LoadInt32(&w.quota) > 0 {
            atomic.AddInt32(&w.quota, -sz)
            return nil  // 配额足够
       	}
        // 等待配额恢复
        <-w.ch
    }
}

// inFlow — Stream 级入站流量跟踪
type inFlow struct {
    pendingData  int32  // 已接收未消费的数据
    pendingUpdate int32  // 待发送的 WINDOW_UPDATE 增量
    limit        int32  // 当前窗口大小
}
```

### BDP 估算器：动态调整窗口

`internal/transport/bdp_estimator.go` 实现了带宽延迟积（BDP）估算器，动态调整流控窗口：

```
原理:
1. 测量一次 RTT（通过 PING 帧）
2. 测量这段时间内传输的字节数
3. BDP = 带宽 × RTT
4. 如果 BDP > 当前窗口，增大窗口
5. 目标: 让管道始终充满数据

例如:
  带宽 = 100 MB/s, RTT = 10ms
  BDP = 100MB × 0.01s = 1MB
  初始窗口 64KB 太小 → 调整为 1MB
```

### SendMsg 中的背压

`stream.go` 中 `SendMsg` 的关键行为：

> SendMsg blocks until:
> - There is sufficient flow control to schedule m with the transport, or
> - The stream is done, or
> - The stream breaks.

这意味着：如果服务端处理慢、窗口耗尽，客户端的 `SendMsg` 会自动阻塞——这就是 gRPC 的背压机制。

---

## 6.7 源码追踪：loopyWriter

`internal/transport/controlbuf.go` 中的 `loopyWriter` 是写调度器——所有 HTTP/2 帧的写入都经过它序列化：

```go
// controlbuf.go（简化版）
type loopyWriter struct {
    side      string             // "client" or "server"
    cbuf      *controlBuffer     // 待发送帧的优先级队列
    hBuf      *bytes.Buffer      // HPACK 编码缓冲区
    hEnc      *hpack.Encoder     // HPACK 编码器
    outbound  map[uint32]*outStream  // 活跃的出站 Stream
}

func (l *loopyWriter) run() {
    for {
        // 从控制缓冲区取出下一个待发送项
        it := l.cbuf.get()
        switch t := it.(type) {
        case *dataFrame:
            // 发送 DATA 帧
            // 1. 检查流控窗口
            // 2. 检查写配额
            // 3. 发送数据（可能分片）
            l.processData(t)
        case *headerFrame:
            // 发送 HEADERS 帧
            l.processHeader(t)
        case *settingsFrame:
            // 发送 SETTINGS 帧
        case *windowUpdate:
            // 发送 WINDOW_UPDATE 帧
        case *ping:
            // 发送 PING 帧
        case *goAway:
            // 发送 GOAWAY 帧
        }
    }
}
```

**loopyWriter 的关键设计**：
- 单协程写：避免 HTTP/2 帧交错问题
- 优先级调度：SETTINGS > PING > HEADERS > DATA
- 流控集成：发送 DATA 前检查窗口和写配额

---

## 6.8 动手实验

### 实验：双向流聊天

```go
// chat.proto
syntax = "proto3";
package chat;
option go_package = "chat";

message ChatMessage {
    string user = 1;
    string text = 2;
}

service Chat {
    rpc ChatStream (stream ChatMessage) returns (stream ChatMessage);
}
```

```go
// 服务端
func (s *chatServer) ChatStream(stream pb.Chat_ChatStreamServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        fmt.Printf("[%s]: %s\n", msg.User, msg.Text)

        // 回显消息
        stream.Send(&pb.ChatMessage{
            User: "server",
            Text: "echo: " + msg.Text,
        })
    }
}
```

```go
// 客户端
func main() {
    conn, _ := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    defer conn.Close()

    stream, _ := pb.NewChatClient(conn).ChatStream(context.Background())

    // 发送协程
    go func() {
        scanner := bufio.NewScanner(os.Stdin)
        for scanner.Scan() {
            stream.Send(&pb.ChatMessage{User: "client", Text: scanner.Text()})
        }
        stream.CloseSend()
    }()

    // 接收循环
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatalf("recv error: %v", err)
        }
        fmt.Printf("[%s]: %s\n", msg.User, msg.Text)
    }
}
```

### 实验：观察 Flow Control 窗口变化

```bash
# 启动服务端（添加 sleep 模拟慢处理）
# 启动客户端，发送大量数据
# 用 Wireshark 抓包，过滤 http2.window_update
# 观察:
# 1. 初始窗口大小
# 2. DATA 帧导致窗口减小
# 3. WINDOW_UPDATE 帧恢复窗口
# 4. 窗口为 0 时 DATA 帧停止
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| 四种模式 | HTTP/2 层面都是 Stream，区别在消息数量和 END_STREAM 时机 |
| Unary 也是 Stream | invoke() 底层创建 ClientStream，发一个收一个 |
| 流式发送/接收 | SendMsg/RecvMsg 可并发调用（不同方向） |
| Flow Control | 连接级 + Stream 级双层窗口，防止压垮接收方 |
| 背压 | 窗口耗尽时 SendMsg 自动阻塞 |
| loopyWriter | 单协程写调度器，优先级排序，流控集成 |
| BDP 估算 | 动态调整窗口大小以匹配网络带宽 |

---

## 下节预告

第7课将深入 **拦截器（Interceptor）与中间件模式**，理解 gRPC 的可扩展性核心机制。
