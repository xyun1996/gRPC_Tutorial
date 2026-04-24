# 第3课：HTTP/2 与 gRPC 帧协议

## 学习目标

- 理解 HTTP/2 的核心概念：Stream、Frame、Flow Control
- 掌握 gRPC 如何映射到 HTTP/2
- 理解 gRPC 帧格式与 Trailers 机制
- 能直接构造和解析 gRPC 帧

---

## 3.1 HTTP/2 核心概念

### HTTP/1.1 的问题

```
HTTP/1.1 的请求-响应模型（线头阻塞）:

连接1: ───[请求A]───[等待]───[响应A]───[请求B]───[等待]───[响应B]───
               ▲ 必须等 A 完成才能发 B

解决方案1: 多个 TCP 连接（浏览器通常 6 个）
  问题: 连接建立开销大，各自独立拥塞控制

解决方案2: HTTP/2 多路复用（一个连接，多个并发流）
```

### HTTP/2 的三大核心

**1. Stream（流）**
```
一个 TCP 连接上可以承载多个并发 Stream
每个 Stream 有唯一 ID（客户端发起的为奇数：1, 3, 5...）
一个 Stream 代表一次请求-响应（在 gRPC 中 = 一次 RPC）
```

**2. Frame（帧）**
```
HTTP/2 通信的最小单位，所有数据都通过帧传输

┌──────────────────────────────────────────────────────┐
│                  HTTP/2 Frame 格式                    │
├──────────┬──────┬──────┬──────────────┬──────────────┤
│ Length   │Type  │Flags │Stream ID     │Payload       │
│ (3字节)  │(1B)  │(1B)  │(4字节)       │(Length字节)  │
└──────────┴──────┴──────┴──────────────┴──────────────┘

帧类型:
  0x0  DATA          - 传输请求/响应体
  0x1  HEADERS       - 传输请求/响应头
  0x2  PRIORITY      - 流优先级
  0x3  RST_STREAM    - 终止流
  0x4  SETTINGS      - 连接参数协商
  0x5  PUSH_PROMISE  - 服务端推送
  0x6  PING          - 保活/RTT 测量
  0x7  GOAWAY        - 优雅关闭
  0x8  WINDOW_UPDATE - 流量控制窗口更新
  0x9  CONTINUATION  - 头部帧延续
```

**3. Flow Control（流量控制）**
```
每个 Stream 和整个连接都有流量控制窗口
发送方不能发送超过窗口大小的数据
接收方通过 WINDOW_UPDATE 帧增加窗口

初始窗口大小: 65535 字节 (defaultWindowSize in defaults.go)

发送方:  available_window -= data_length
接收方:  发送 WINDOW_UPDATE{increment} 恢复窗口
```

### HTTP/2 连接建立过程

```
客户端                                    服务端
  │                                         │
  │─── TCP 三次握手 ───────────────────────►│
  │                                         │
  │─── HTTP/2 连接前言 (Magic + SETTINGS) ─►│
  │    "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"   │
  │    + SETTINGS 帧                         │
  │                                         │
  │◄── SETTINGS 帧 ────────────────────────│
  │                                         │
  │─── SETTINGS ACK ───────────────────────►│
  │◄── SETTINGS ACK ───────────────────────│
  │                                         │
  │     连接建立完成，可以开始发 RPC          │
```

> 源码参考：`internal/transport/http2_client.go` 中 `newHTTP2Client()` 函数实现了完整的连接建立过程。

---

## 3.2 gRPC 如何映射到 HTTP/2

### gRPC 请求-响应的 HTTP/2 映射

gRPC 规范定义了 RPC 如何映射到 HTTP/2：

```
gRPC 请求 = HTTP/2 HEADERS + DATA
gRPC 响应 = HTTP/2 HEADERS + DATA + TRAILERS

完整映射关系:

┌──────────────┬─────────────────────────────────────┐
│  gRPC 概念    │  HTTP/2 映射                         │
├──────────────┼─────────────────────────────────────┤
│ RPC 方法     │ :path = /package.service/method      │
│ 请求元数据    │ HEADERS 帧 (HTTP/2 头部)             │
│ 请求消息      │ DATA 帧 (Length-Prefixed-Message)   │
│ 响应元数据    │ HEADERS 帧                          │
│ 响应消息      │ DATA 帧                             │
│ RPC 状态      │ TRAILERS 帧 (grpc-status)           │
│ 流式 RPC     │ 同一 Stream ID 上多个 DATA 帧         │
│ 取消 RPC     │ RST_STREAM 帧                       │
└──────────────┴─────────────────────────────────────┘
```

### gRPC 请求头 (Request Headers)

一次 Unary RPC 的请求头：

```
HEADERS 帧 (Stream ID = 1, END_STREAM=false):

:method = POST
:scheme = http 或 https
:path = /helloworld.Greeter/SayHello     ← 关键：标识 RPC 方法
:authority = localhost:50051              ← 虚拟主机
content-type = application/grpc           ← gRPC 标识
te = trailers                             ← 声明需要 trailers
grpc-encoding = identity                  ← 消息编码 (identity/gzip)
grpc-accept-encoding = identity,gzip      ← 接受的编码
user-agent = grpc-go/1.xx                 ← 客户端标识
grpc-timeout = 1S                         ← 超时（可选）
```

### 源码：请求头的构造

请求头在 `internal/transport/http2_client.go` 的 `NewStream()` 方法中构造：

```go
// 简化版，展示核心逻辑
func (t *http2Client) NewStream(ctx context.Context, callHdr *CallHdr) (*Stream, error) {
    // 构造 HTTP/2 头部
    hdr := &headerFieldList{
        {":method", "POST"},
        {":path", callHdr.Method},           // /package.service/method
        {":authority", callHdr.Host},         // 目标地址
        {"content-type", "application/grpc"},
        {"te", "trailers"},                   // 声明使用 trailers
    }

    // 添加用户自定义元数据
    for k, vv := range md {
        for _, v := range vv {
            hdr = append(hdr, headerField{k, v})
        }
    }

    // 创建 HTTP/2 Stream
    stream := &Stream{
        id:   t.nextID(),                     // 奇数递增: 1, 3, 5...
        method: callHdr.Method,
    }

    // 通过 loopyWriter 发送 HEADERS 帧
    t.controlBuf.executeAndPut(func() {
        t.loopyWriter.writeHeader(stream.id, hdr)
    })

    return stream, nil
}
```

---

## 3.3 gRPC 帧格式：Length-Prefixed-Message

### 帧格式定义

gRPC 在 HTTP/2 DATA 帧中的消息格式：

```
┌────────────┬────────────┬─────────────────────┐
│ Compressed  │  Message   │     Message         │
│ Flag (1B)   │  Length    │     Data            │
│             │  (4字节)   │     (Length字节)    │
└────────────┴────────────┴─────────────────────┘

Compressed Flag:
  0 = 未压缩 (identity)
  1 = 已压缩 (gzip 等)

Message Length: 4 字节大端序无符号整数
Message Data:   Protobuf 编码后的字节流
```

### 源码中的帧解析

帧的读取逻辑在 `internal/transport/transport.go` 的 `recvBufferReader.ReadMessageHeader()` 方法中：

```go
// ReadMessageHeader 读取 5 字节的消息头
// 1 字节压缩标志 + 4 字节消息长度
func (r *recvBufferReader) ReadMessageHeader(header []byte) (n int, err error) {
    // header 是 5 字节的缓冲区
    // header[0]    = Compressed-Flag
    // header[1:5]  = Message-Length (big-endian uint32)
    ...
}
```

消息写入的封装逻辑（简化版）：

```go
// 写入 gRPC 消息帧
func writeMsg(frame []byte, msg []byte, compressed bool) {
    // 1. 压缩标志
    if compressed {
        frame[0] = 1
    } else {
        frame[0] = 0
    }

    // 2. 消息长度（4 字节大端序）
    binary.BigEndian.PutUint32(frame[1:5], uint32(len(msg)))

    // 3. 消息体
    copy(frame[5:], msg)
}
```

### 完整的 Unary RPC 帧交换

```
客户端                                    服务端
  │                                         │
  │─── HEADERS (Stream 1) ────────────────►│
  │    :method: POST                        │
  │    :path: /helloworld.Greeter/SayHello  │
  │    content-type: application/grpc       │
  │    te: trailers                         │
  │                                         │
  │─── DATA (Stream 1) ──────────────────►│
  │    [0x00][0x00 0x00 0x00 0x07]         │
  │    [Protobuf 编码的 HelloRequest]        │
  │    Compressed=0, Length=7               │
  │                                         │
  │◄── HEADERS (Stream 1) ─────────────────│
  │    :status: 200                         │
  │    content-type: application/grpc       │
  │                                         │
  │◄── DATA (Stream 1) ───────────────────│
  │    [0x00][0x00 0x00 0x00 0x0C]         │
  │    [Protobuf 编码的 HelloReply]          │
  │    Compressed=0, Length=12              │
  │                                         │
  │◄── HEADERS (Stream 1, END_STREAM) ─────│
  │    grpc-status: 0                       │ ← Trailers
  │    grpc-message: (空)                    │
  │                                         │
  │  RPC 完成                                │
```

---

## 3.4 Trailers 机制与 gRPC Status 传递

### 为什么需要 Trailers？

HTTP/1.1 只在响应开头发送头部，无法在响应体之后添加信息。
HTTP/2 引入了 **Trailers**——在 DATA 帧之后发送的额外头部帧，用于传递响应的最终状态。

```
HTTP/2 响应结构:

HEADERS ──► DATA ──► HEADERS (Trailers)
  │           │          │
  │           │          └── gRPC 状态码和消息
  │           └──────────── 响应消息体
  └──────────────────────── 响应初始头部
```

这是 `te: trailers` 请求头的意义——客户端声明"我理解 trailers"。

### gRPC Status 的 HTTP/2 编码

```
Trailers HEADERS 帧 (END_STREAM=true):

成功:
  grpc-status: 0                ← OK
  grpc-message: (空)

失败:
  grpc-status: 12               ← UNIMPLEMENTED
  grpc-message: method not found

带详情的失败:
  grpc-status: 3                ← INVALID_ARGUMENT
  grpc-message: invalid field
  grpc-status-details-bin: Cj4... (Base64 编码的 google.rpc.Status proto)
```

### 源码中的状态码映射

`internal/transport/http_util.go` 定义了 HTTP/2 错误码到 gRPC 状态码的映射：

```go
var http2ErrConvTab = map[http2.ErrCode]codes.Code{
    http2.ErrCodeNo:                 codes.Internal,
    http2.ErrCodeProtocol:           codes.Internal,
    http2.ErrCodeFlowControl:        codes.ResourceExhausted,
    http2.ErrCodeRefusedStream:      codes.Unavailable,
    http2.ErrCodeCancel:             codes.Canceled,
    http2.ErrCodeEnhanceYourCalm:    codes.ResourceExhausted,
    http2.ErrCodeInadequateSecurity: codes.PermissionDenied,
}

// HTTP 状态码到 gRPC 状态码的映射
var HTTPStatusConvTab = map[int]codes.Code{
    http.StatusBadRequest:          codes.Internal,
    http.StatusUnauthorized:         codes.Unauthenticated,
    http.StatusForbidden:            codes.PermissionDenied,
    http.StatusNotFound:             codes.Unimplemented,
    http.StatusTooManyRequests:      codes.Unavailable,
    http.StatusBadGateway:           codes.Unavailable,
    http.StatusServiceUnavailable:   codes.Unavailable,
}
```

### Stream 生命周期状态

在 `internal/transport/transport.go` 中，Stream 有四个状态：

```go
const (
    streamActive    streamState = iota  // Stream 活跃
    streamWriteDone                     // 已发送 EndStream
    streamReadDone                      // 已接收 EndStream
    streamDone                          // Stream 完全结束
)
```

对应 gRPC 规范的 Stream 生命周期：

```
streamActive ──► streamWriteDone ──► streamDone
     │                                    ▲
     └────► streamReadDone ───────────────┘

streamWriteDone: 客户端调用 CloseSend() 或服务端发送完最后一个消息
streamReadDone:  收到对方发来的 END_STREAM 标志
streamDone:      两端都完成
```

---

## 3.4-S 客户端 Stream ID 分配机制

### 初始值和递增规则

HTTP/2 规范（RFC 7540, Section 5.1.1）规定：**客户端发起的流必须用奇数 ID，服务端发起的用偶数 ID**。gRPC-Go 严格遵循：

```go
// http2_client.go:357 — 客户端 nextID 初始值为 1
nextID: 1,

// http2_client.go:857-858 — 每次创建新流时
hdr.streamID = t.nextID   // 使用当前值：1, 3, 5, 7...
t.nextID += 2             // 递增 2，保持奇数
```

所以客户端发出的 stream ID 序列是：**1 → 3 → 5 → 7 → 9 → ...**

### 为什么是奇数？

收到一个 HEADERS 帧时，只需看 stream ID 的奇偶性就能判断来源，无需额外标识。服务端也做了对应校验：

```go
// http2_server.go:395
if streamID%2 != 1 || streamID <= t.maxStreamID {
    // illegal gRPC stream id.
}
```

---

## 3.4-T HTTP/2 连接建立的源码全流程

### 调用链

```
addrConn.resetTransportAndUnlock()        // clientconn.go:1326
  └─ ac.tryAllAddrs()                     // clientconn.go:1452
       └─ ac.createTransport()            // clientconn.go:1490
            └─ transport.NewHTTP2Client() // http2_client.go:210
```

### NewHTTP2Client 内部步骤

| 步骤 | 源码位置 | 作用 |
|------|---------|------|
| TCP 拨号 | `http2_client.go:225` `dial()` | 建立 TCP 连接（支持自定义 Dialer、代理） |
| TLS 握手 | `http2_client.go:295` `ClientHandshake()` | TLS 1.3/1.2 握手（如有 transport creds） |
| 写前言 | `http2_client.go:431` | 发送 `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` |
| 写 SETTINGS | `http2_client.go:454` | 发送初始 SETTINGS（窗口大小等） |
| 可能 WINDOW_UPDATE | `http2_client.go:460` | 如果自定义了连接窗口大小 |
| 启动 reader | `http2_client.go:420` `go t.reader()` | 后台读帧协程 |
| 等待握手完成 | `http2_client.go:473` `<-readerErrCh` | 阻塞直到收到服务端 SETTINGS |
| 启动 loopy writer | `http2_client.go:477` `go t.loopy.run()` | 后台写帧协程 |

### reader 协程的工作

```go
// http2_client.go:1655
func (t *http2Client) reader(errCh chan<- error) {
    // 1. 读取服务端 SETTINGS 帧（连接握手）
    t.readServerPreface()          // → errCh <- nil 表示握手成功

    // 2. 进入读循环
    for {
        frame := t.framer.readFrame()
        // 根据帧类型分发处理:
        //   DATA        → 写入对应 stream 的接收缓冲区
        //   HEADERS     → 处理响应头或 trailers
        //   SETTINGS    → 更新连接参数
        //   PING        → 回复 ACK
        //   GOAWAY      → 触发优雅关闭
        //   RST_STREAM  → 终止对应 stream
    }
}
```

### 时序图

```
Client                              Server
  │──── TCP SYN ────────────────────→  │
  │←─── TCP SYN+ACK ────────────────  │
  │──── TCP ACK ───────────────────→  │   ← dial()
  │                                    │
  │──── TLS ClientHello ────────────→  │
  │←─── TLS ServerHello + Cert ─────  │
  │──── TLS Finished ───────────────→  │   ← ClientHandshake()
  │←─── TLS Finished ───────────────  │
  │                                    │
  │──── PRI * HTTP/2.0\r\n\r\n ──────→  │   ← 客户端前言
  │──── SETTINGS[...] ──────────────→  │   ← 客户端参数
  │                                    │
  go reader()                          │   ← 启动读协程
  │                                    │
  │←─── SETTINGS[...] ──────────────  │   ← 服务端前言+参数
  │                                    │
  │──── SETTINGS ACK ───────────────→  │   ← loopy writer 自动回复
  │←─── SETTINGS ACK ───────────────  │
  │                                    │
  go loopy.run()                       │   ← 启动写协程
  │                                    │
  │======== HTTP/2 连接就绪 ==========│
```

### 关键源码位置速查

| 文件 | 函数 | 职责 |
|------|------|------|
| `clientconn.go:1326` | `resetTransportAndUnlock` | 触发连接 |
| `clientconn.go:1452` | `tryAllAddrs` | 逐个地址尝试 |
| `clientconn.go:1490` | `createTransport` | 创建单个传输 |
| `http2_client.go:161` | `dial()` | TCP 拨号 |
| `http2_client.go:295` | `ClientHandshake()` | TLS 握手 |
| `http2_client.go:431` | Write clientPreface | 写 HTTP/2 前言 |
| `http2_client.go:1639` | `readServerPreface` | 读服务端 SETTINGS |
| `http2_client.go:1655` | `reader()` | 读帧循环 |
| `http2_client.go:477` | `loopy.run()` | 写帧循环 |

---

## 3.5 动手实验

### 实验1：用 Go 直接发送 HTTP/2 帧模拟 gRPC 请求

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net"
    "net/http"
    "strings"

    "golang.org/x/net/http2"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
    "google.golang.org/protobuf/proto"
)

func main() {
    // 1. 建立 TCP 连接
    conn, err := net.Dial("tcp", "localhost:50051")
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    // 2. 手动发送 HTTP/2 连接前言
    // 这是 HTTP/2 规范规定的客户端魔法字符串
    preface := "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
    conn.Write([]byte(preface))

    // 3. 使用 http2.Transport 发送请求（更实用的方式）
    client := &http.Client{
        Transport: &http2.Transport{
            AllowHTTP: true, // 允许明文 HTTP/2 (h2c)
            DialTLSContext: func(ctx context.Context, network, addr string, _ *tls.Config) (net.Conn, error) {
                return net.Dial(network, addr)
            },
        },
    }

    // 4. 手动构造 gRPC 请求帧
    req := &pb.HelloRequest{Name: "world"}
    payload, _ := proto.Marshal(req)

    // 构造 Length-Prefixed-Message
    // 1 字节压缩标志 + 4 字节长度 + Protobuf 数据
    msg := make([]byte, 5+len(payload))
    msg[0] = 0 // 未压缩
    msg[1] = byte(len(payload) >> 24)
    msg[2] = byte(len(payload) >> 16)
    msg[3] = byte(len(payload) >> 8)
    msg[4] = byte(len(payload))
    copy(msg[5:], payload)

    // 5. 发送 HTTP/2 POST 请求
    httpReq, _ := http.NewRequest("POST",
        "http://localhost:50051/helloworld.Greeter/SayHello",
        strings.NewReader(string(msg)))
    httpReq.Header.Set("Content-Type", "application/grpc")
    httpReq.Header.Set("Te", "trailers")

    resp, err := client.Do(httpReq)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    // 6. 读取 gRPC 响应帧
    respBody, _ := io.ReadAll(resp.Body)

    if len(respBody) >= 5 {
        // 解析 Length-Prefixed-Message
        compressed := respBody[0]
        msgLen := uint32(respBody[1])<<24 | uint32(respBody[2])<<16 |
            uint32(respBody[3])<<8 | uint32(respBody[4])

        fmt.Printf("Compressed: %d\n", compressed)
        fmt.Printf("Message Length: %d\n", msgLen)

        // 解码 Protobuf 响应
        reply := &pb.HelloReply{}
        proto.Unmarshal(respBody[5:5+msgLen], reply)
        fmt.Printf("Response: %s\n", reply.GetMessage())
    }

    // 7. 读取 Trailers (gRPC Status)
    fmt.Printf("gRPC Status: %s\n", resp.Trailer.Get("grpc-status"))
    fmt.Printf("gRPC Message: %s\n", resp.Trailer.Get("grpc-message"))
}
```

### 实验2：观察 gRPC 帧的大小限制

```go
package main

import (
    "context"
    "fmt"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/encoding/gzip"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithDefaultCallOptions(
            grpc.UseCompressor(gzip.Name), // 启用 gzip 压缩
        ),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)

    // 开启 gRPC 调试日志观察帧
    resp, err := client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "compressed world"})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Printf("Greeting: %s\n", resp.GetMessage())

    // 在日志中观察:
    // - 请求帧中 Compressed-Flag = 1
    // - grpc-encoding = gzip
    // - 消息体比未压缩小
}
```

### 实验3：观察 HTTP/2 SETTINGS 协商

```bash
# 开启 gRPC 详细日志
export GRPC_GO_LOG_VERBOSITY_LEVEL=99
export GRPC_GO_LOG_SEVERITY_LEVEL=info

# 运行客户端
go run examples/helloworld/greeter_client/main.go
```

日志中会看到 SETTINGS 帧的交换：
```
INFO: [transport] client connection created
INFO: [transport] Received SETTINGS frame: ...
# 关键 SETTINGS 参数:
# SETTINGS_MAX_CONCURRENT_STREAMS = 100    (最大并发流数)
# SETTINGS_INITIAL_WINDOW_SIZE = 65535     (初始流控窗口)
# SETTINGS_MAX_FRAME_SIZE = 16384          (最大帧大小 = 16KB)
# SETTINGS_HEADER_TABLE_SIZE = 4096        (HPACK 头部表大小)
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| HTTP/2 Stream | 一个 TCP 连接上的逻辑通道，用 Stream ID 标识 |
| HTTP/2 Frame | 通信最小单位：DATA、HEADERS、SETTINGS、WINDOW_UPDATE 等 |
| Flow Control | 基于窗口的流量控制，防止发送方压垮接收方 |
| gRPC 映射 | RPC 方法→`:path`，请求→HEADERS+DATA，状态→Trailers |
| Length-Prefixed-Message | 5 字节头(1B标志+4B长度) + Protobuf 数据 |
| Trailers | DATA 之后的 HEADERS 帧，传递 grpc-status |

---

## 下节预告

第4课将深入 **Channel 与连接管理**，理解 ClientConn 的状态机、Resolver 如何工作，以及连接是如何建立和复用的。
