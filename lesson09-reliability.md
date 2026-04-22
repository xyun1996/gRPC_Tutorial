# 第9课：可靠性机制：重试、超时与死线传播

## 学习目标

- 掌握 Deadline 与超时传播机制
- 理解重试策略：透明重试 vs 应用层重试
- 掌握 Hedging（对冲）策略
- 理取消传播机制

---

## 9.1 Deadline 与超时传播

### Deadline 的概念

gRPC 的 Deadline 是一个**绝对时间点**——超过此时间，RPC 将被取消。与之相对的是 Timeout（相对时间），gRPC 在网络上传输的是超时值。

```go
// 设置 Deadline
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "world"})
if err != nil {
    code := status.Code(err)
    // DeadlineExceeded 时 code == codes.DeadlineExceeded
}
```

### grpc-timeout 头的逐跳传播

gRPC 超时通过 HTTP/2 头部 `grpc-timeout` 传播，**每一跳都会递减**：

```
客户端                 服务A                服务B
  │── grpc-timeout: 5S ──►│                    │
  │   (5秒超时)            │── grpc-timeout: 3S ──►│
  │                        │   (减去A的处理时间2s)   │
  │                        │                    │── grpc-timeout: 1S ──►服务C
  │                        │                    │   (减去B的处理时间2s)
```

### grpc-timeout 编码格式

```
grpc-timeout = TimeoutValue TimeoutUnit

TimeoutUnit:
  H  = 小时
  M  = 分钟
  S  = 秒
  m  = 毫秒
  u  = 微秒
  n  = 纳秒

示例:
  grpc-timeout: 5S    → 5 秒
  grpc-timeout: 300m  → 300 毫秒
  grpc-timeout: 5000u → 5000 微秒
```

### 源码：超时的设置与传播

```go
// stream.go — newClientStream 中设置超时
func newClientStream(...) (*clientStream, error) {
    // 从 context 获取 deadline
    if dl, ok := ctx.Deadline(); ok {
        // 计算剩余超时
        timeout := time.Until(dl)
        if timeout <= 0 {
            return nil, status.Error(codes.DeadlineExceeded, "context deadline exceeded")
        }
        // 将超时编码到 grpc-timeout 头部
        md = md.Copy()
        md.Set("grpc-timeout", encodeTimeout(timeout))
    }
    // ...
}
```

---

## 9.2 重试策略

### 透明重试（Transparent Retry）

gRPC 框架内置的重试——**无需任何配置**：

```
何时触发透明重试:
1. 连接在收到服务端任何响应之前断开
2. 服务端返回 UNAVAILABLE 错误
3. RPC 未到达服务端应用层

不触发透明重试的情况:
1. 服务端已返回任何响应头
2. 超过 maxAttempts（默认 1）
3. 调用了 Header() 方法（禁用重试）
```

### 应用层重试（配置驱动）

通过 ServiceConfig 配置重试策略：

```json
{
  "methodConfig": [{
    "name": [{"service": "example.ExampleService"}],
    "retryPolicy": {
      "maxAttempts": 3,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2.0,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}
```

### 源码：重试实现

`stream.go` 中的 `newClientStreamWithRetries`：

```go
func newClientStreamWithRetries(ctx context.Context, ...) (*clientStream, error) {
    // 获取重试策略
    methodConfig := cc.methodConfig(method)
    retryPolicy := methodConfig.RetryPolicy

    if retryPolicy == nil || retryPolicy.MaxAttempts == 1 {
        // 无重试策略，直接调用
        return newClientStream(ctx, ...)
    }

    // 重试循环
    for attempt := 0; attempt < retryPolicy.MaxAttempts; attempt++ {
        cs, err := newClientStream(ctx, ...)

        if attempt == 0 {
            // 第一次尝试，如果是连接级别错误，执行透明重试
            if isRetryableError(err) {
                continue
            }
        }

        // 检查返回的状态码是否可重试
        if cs.attempt.status != nil {
            code := cs.attempt.status.Code()
            for _, retryable := range retryPolicy.RetryableStatusCodes {
                if code == retryable {
                    // 计算退避时间
                    backoff := calculateBackoff(
                        retryPolicy.InitialBackoff,
                        retryPolicy.MaxBackoff,
                        retryPolicy.BackoffMultiplier,
                        attempt,
                    )
                    time.Sleep(backoff)
                    continue
                }
            }
        }

        return cs, err
    }
}
```

### retryThrottler：重试限流

防止重试风暴——当错误率高时自动减少重试：

```go
// clientconn.go
type retryThrottler struct {
    max    float64  // 阈值（默认 0.8）
    thresh float64  // 触发限流的阈值
    ratio  float64  // 令牌比率（成功增加，失败减少）
    mu     sync.Mutex
}

func (rt *retryThrottler) shouldRetry() bool {
    rt.mu.Lock()
    defer rt.mu.Unlock()
    // 当 token < thresh 时，拒绝重试
    return rt.thresh >= rt.max || rt.thresh < rt.max
}
```

**工作机制**：
- 每次 RPC 成功 → token += ratio（如 0.1）
- 每次 RPC 失败 → token -= 1.0
- 当 token < thresh → 停止重试
- 自适应：错误率高时减少重试，成功率高时恢复

---

## 9.3 Hedging（对冲）策略

Hedging 与 Retry 的区别：

```
Retry（重试）:
  请求1 ──失败──► 等待退避 ──► 请求2 ──失败──► 等待 ──► 请求3
  串行，有延迟

Hedging（对冲）:
  请求1 ──────────────────────►
  ───等待HedgingDelay──► 请求2 ──►
  ───等待HedgingDelay──► 请求3 ──►
  并行，哪个先成功就用哪个
```

### Hedging 配置

```json
{
  "methodConfig": [{
    "name": [{"service": "example.ExampleService"}],
    "hedgingPolicy": {
      "maxAttempts": 3,
      "hedgingDelay": "100ms",
      "nonFatalStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}
```

### Hedging 的工作流程

```
t=0ms:    发送 Hedging 请求 1
t=100ms:  请求1 还没响应 → 发送 Hedging 请求 2
t=200ms:  请求1、2 都没响应 → 发送 Hedging 请求 3

任何请求返回成功 → 取消其他请求，返回成功
任何请求返回非 nonFatal 错误 → 取消其他请求，返回错误
所有请求都返回错误 → 返回最后一个错误
```

### 幂等性要求

- **Retry** 要求方法**安全**（Safe）：重试不会产生副作用
- **Hedging** 要求方法**幂等**（Idempotent）：多个并发请求等价于一个请求

```
安全的方法: 读取数据（GetFeature）
幂等的方法: 设置相同值（SetValue(5)）、递增（Increment() 不幂等！）
```

---

## 9.4 Backoff 策略

### 指数退避

```go
// backoff/.go 中的退避计算
func Backoff(attempt int, config backoff.Config) time.Duration {
    // 基础退避 = initialBackoff * multiplier^attempt
    backoff := float64(config.InitialBackoff)
    for i := 0; i < attempt; i++ {
        backoff *= config.MaxBackoff
        if backoff > float64(config.MaxBackoff) {
            backoff = float64(config.MaxBackoff)
            break
        }
    }
    // 加入抖动（Jitter）防止惊群
    backoff *= 0.8 + rand.Float64()*0.4  // ±20% 抖动
    return time.Duration(backoff)
}
```

```
重试时间线（initialBackoff=100ms, maxBackoff=1s, multiplier=2）:

尝试1: 失败 → 等待 ~100ms  (100ms × 1 ± 20%)
尝试2: 失败 → 等待 ~200ms  (100ms × 2 ± 20%)
尝试3: 失败 → 等待 ~400ms  (100ms × 4 ± 20%)
尝试4: 失败 → 等待 ~800ms  (100ms × 8 ± 20%)
尝试5: 失败 → 等待 ~1s     (达到 maxBackoff)
```

---

## 9.5 取消传播

### 客户端取消如何传播到服务端

```
客户端                           服务端
  │── HEADERS ────────────────►│
  │── DATA ───────────────────►│
  │                            │
  │  ctx.Cancel()              │
  │── RST_STREAM(stream_id) ─►│  ← 取消信号
  │                            │
  │                     服务端收到 RST_STREAM:
  │                     1. 停止处理该 RPC
  │                     2. 关闭 Stream
  │                     3. 释放资源
```

### 源码：取消的传播

```go
// internal/transport/http2_client.go
// 当 context 被取消时
func (t *http2Client) closeStream(s *Stream, err error) {
    // 发送 RST_STREAM 帧通知服务端
    if s.id != 0 {
        t.controlBuf.executeAndPut(func() {
            t.framer.writeRSTStream(s.id, http2.ErrCodeCancel)
        })
    }
    // 关闭 Stream 的接收缓冲区
    s.recvBuffer.put(recvMsg{err: err})
}
```

### 服务端检测取消

```go
func (s *server) LongRunningRPC(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    // 方式1: 检查 context
    select {
    case <-ctx.Done():
        return nil, status.Error(codes.Canceled, "request canceled")
    default:
    }

    // 方式2: 在循环中检查
    for _, item := range items {
        if ctx.Err() != nil {
            return nil, status.FromContextError(ctx.Err()).Err()
        }
        process(item)
    }

    return &pb.Response{}, nil
}
```

---

## 9.6 动手实验：模拟网络故障，观察重试行为

### 实验：配置重试策略

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    // 配置重试策略
    retryServiceConfig := `{
        "methodConfig": [{
            "name": [{"service": "helloworld.Greeter"}],
            "retryPolicy": {
                "maxAttempts": 4,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2.0,
                "retryableStatusCodes": ["UNAVAILABLE"]
            }
        }]
    }`

    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(retryServiceConfig),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)

    // 测试1: 正常调用
    resp, err := client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatalf("error: %v", err)
    }
    fmt.Printf("Response: %s\n", resp.GetMessage())

    // 测试2: 超时
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    _, err = client.SayHello(ctx, &pb.HelloRequest{Name: "timeout"})
    if err != nil {
        code := status.Code(err)
        fmt.Printf("Timeout test: code=%s message=%v\n", code, err)
    }

    // 测试3: 主动取消
    ctx, cancel = context.WithCancel(context.Background())
    go func() {
        time.Sleep(50 * time.Millisecond)
        cancel()  // 50ms 后取消
    }()
    _, err = client.SayHello(ctx, &pb.HelloRequest{Name: "cancel"})
    if err != nil {
        code := status.Code(err)
        fmt.Printf("Cancel test: code=%s message=%v\n", code, err)
    }
}
```

### 实验：模拟间歇性故障的服务端

```go
package main

import (
    "context"
    "log"
    "math/rand"
    "net"
    "sync/atomic"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

type flakyServer struct {
    pb.UnimplementedGreeterServer
    callCount atomic.Int32
}

func (s *flakyServer) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    count := s.callCount.Add(1)
    log.Printf("Call #%d: %s", count, req.Name)

    // 50% 概率返回 UNAVAILABLE
    if rand.Float32() < 0.5 {
        log.Printf("Call #%d: returning UNAVAILABLE", count)
        return nil, status.Error(codes.Unavailable, "service temporarily unavailable")
    }

    return &pb.HelloReply{Message: "Hello " + req.Name}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &flakyServer{})
    log.Println("Flaky server listening on :50051")
    s.Serve(lis)
}
```

> 运行上述客户端和服务端，观察日志中的重试行为：
> - 客户端第一次 UNAVAILABLE → 等待退避 → 重试 → 可能成功
> - 最多重试 4 次（1 次原始 + 3 次重试）
> - 如果全部失败，返回最后一个错误

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| Deadline | 绝对时间点，超时传播通过 `grpc-timeout` 头逐跳递减 |
| 透明重试 | 框架内置，无需配置，RPC 未到达应用层时自动重试 |
| 应用层重试 | ServiceConfig 配置，指定可重试状态码和退避策略 |
| Hedging | 并行对冲，哪个先返回用哪个，要求方法幂等 |
| retryThrottler | 自适应限流，错误率高时减少重试 |
| Backoff | 指数退避 + 抖动，防止惊群效应 |
| 取消传播 | RST_STREAM 帧通知服务端，服务端检查 ctx.Err() |

---

## 下节预告

第10课将深入 **高级架构 — xDS、多语言与生产实践**，从全局视角理解 gRPC 的生产级部署。
