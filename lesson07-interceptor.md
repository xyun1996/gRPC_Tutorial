# 第7课：拦截器（Interceptor）与中间件模式

## 学习目标

- 掌握四种拦截器类型的区别与用法
- 理解拦截器链的执行模型与调用顺序
- 能实现常见拦截器：认证、日志、限流、恢复
- 理解拦截器与 gRPC 内部机制的交互

---

## 7.1 拦截器类型体系

### 四种拦截器

gRPC 提供四种拦截器，分别对应客户端/服务端 × 一元/流式：

```
┌──────────────┬──────────────────────┬──────────────────────┐
│              │       一元 RPC        │      流式 RPC         │
├──────────────┼──────────────────────┼──────────────────────┤
│   客户端      │ UnaryClientInterceptor│ StreamClientInterceptor│
│   服务端      │ UnaryServerInterceptor│ StreamServerInterceptor│
└──────────────┴──────────────────────┴──────────────────────┘
```

### 拦截器接口定义

`interceptor.go` 定义了四种拦截器签名：

```go
// 客户端一元拦截器
type UnaryClientInterceptor func(
    ctx context.Context,
    method string,             // RPC 方法名 "/package.service/method"
    req, reply any,            // 请求和响应消息
    cc *ClientConn,            // ClientConn
    invoker UnaryInvoker,      // ★ 调用此函数继续 RPC
    opts ...CallOption,        // 调用选项
) error

// 客户端流拦截器
type StreamClientInterceptor func(
    ctx context.Context,
    desc *StreamDesc,          // 流描述（是否双向流等）
    cc *ClientConn,
    method string,
    streamer Streamer,         // ★ 调用此函数创建 Stream
    opts ...CallOption,
) (ClientStream, error)

// 服务端一元拦截器
type UnaryServerInterceptor func(
    ctx context.Context,
    req any,                   // 请求消息
    info *UnaryServerInfo,     // 服务信息（FullMethod）
    handler UnaryHandler,      // ★ 调用此函数执行业务逻辑
) (resp any, err error)

// 服务端流拦截器
type StreamServerInterceptor func(
    srv any,
    ss ServerStream,           // 服务端流
    info *StreamServerInfo,    // 流信息
    handler StreamHandler,     // ★ 调用此函数执行业务逻辑
) error
```

---

## 7.2 拦截器链的执行模型

### 单个拦截器的执行模型

```
客户端调用: client.SayHello(ctx, req)
  │
  ▼
UnaryClientInterceptor(ctx, method, req, reply, cc, invoker, opts)
  │
  ├── 前置逻辑: 记录开始时间、添加 metadata、检查 token
  │
  ├── 调用 invoker: err := invoker(ctx, method, req, reply, cc, opts...)
  │   │                                         ─────────────────
  │   │                                         这才是真正的 RPC 调用
  │   │
  │   └── invoker 返回
  │
  ├── 后置逻辑: 记录耗时、处理错误、指标采集
  │
  └── 返回 error
```

### 链式拦截器

多个拦截器按注册顺序组成链：

```
调用顺序（洋葱模型）:

  Interceptor1 前置
    └── Interceptor2 前置
          └── Interceptor3 前置
                └── invoker()  ← 实际 RPC
                └── Interceptor3 后置
          └── Interceptor2 后置
    └── Interceptor1 后置
```

### 源码：链式拦截器的组装

`dialoptions.go` 中通过 `WithChainUnaryInterceptor()` 注册链式拦截器：

```go
// dialoptions.go
func WithChainUnaryInterceptor(interceptors ...UnaryClientInterceptor) DialOption {
    return newFuncDialOption(func(o *dialOptions) {
        o.chainUnaryInts = append(o.chainUnaryInts, interceptors...)
    })
}
```

链式组装的核心逻辑（`stream.go` 中的 `chainUnaryClientInterceptors`）：

```go
func chainUnaryClientInterceptors(cc *ClientConn) {
    interceptors := cc.dopts.chainUnaryInts
    // 单个拦截器也加入链中
    if cc.dopts.unaryInt != nil {
        interceptors = append([]UnaryClientInterceptor{cc.dopts.unaryInt},
            interceptors...)
    }

    if len(interceptors) == 0 {
        return  // 无拦截器
    }

    // ★ 递归构建链
    cc.dopts.unaryInt = func(ctx context.Context, method string,
        req, reply any, cc *ClientConn, invoker UnaryInvoker,
        opts ...CallOption) error {

        return interceptors[0](ctx, method, req, reply, cc,
            getChainUnaryInvoker(interceptors, 0, invoker), opts...)
    }
}

// 递归构建 invoker 链
func getChainUnaryInvoker(interceptors []UnaryClientInterceptor,
    curr int, finalInvoker UnaryInvoker) UnaryInvoker {

    if curr == len(interceptors)-1 {
        return finalInvoker  // 最后一个拦截器调用真正的 invoker
    }

    return func(ctx context.Context, method string,
        req, reply any, cc *ClientConn, opts ...CallOption) error {

        return interceptors[curr+1](ctx, method, req, reply, cc,
            getChainUnaryInvoker(interceptors, curr+1, finalInvoker), opts...)
    }
}
```

### 服务端拦截器的注册

```go
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryInterceptor,
        loggingInterceptor,
        authInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        recoveryStreamInterceptor,
        loggingStreamInterceptor,
    ),
)
```

---

## 7.3 常见拦截器实现

### 认证拦截器

```go
func AuthInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

    // 从 metadata 中提取 token
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }

    // 验证 token
    claims, err := validateToken(tokens[0])
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // 将用户信息注入 context
    ctx = context.WithValue(ctx, userKey, claims)

    return handler(ctx, req)
}
```

### 日志拦截器

```go
func LoggingInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

    start := time.Now()

    // 调用业务逻辑
    resp, err := handler(ctx, req)

    // 记录日志
    duration := time.Since(start)
    code := status.Code(err)

    log.Printf("method=%s duration=%v code=%s error=%v",
        info.FullMethod, duration, code, err)

    return resp, err
}
```

### 限流拦截器

```go
func RateLimitInterceptor(limiter *rate.Limiter) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any,
        info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {

        if !limiter.Allow() {
            return nil, status.Error(codes.ResourceExhausted,
                "rate limit exceeded")
        }

        return handler(ctx, req)
    }
}

// 使用
limiter := rate.NewLimiter(rate.Every(time.Second), 100) // 100 QPS
s := grpc.NewServer(grpc.ChainUnaryInterceptor(
    RateLimitInterceptor(limiter),
))
```

### 恢复拦截器（Panic Recovery）

```go
func RecoveryInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {

    defer func() {
        if r := recover(); r != nil {
            // 将 panic 转换为 gRPC 错误
            err = status.Errorf(codes.Internal,
                "panic recovered: %v", r)
            log.Printf("panic in %s: %v\n%s",
                info.FullMethod, r, debug.Stack())
        }
    }()

    return handler(ctx, req)
}
```

### 客户端重试拦截器

```go
func RetryInterceptor(maxRetries int, backoff time.Duration) grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply any,
        cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {

        var lastErr error
        for attempt := 0; attempt <= maxRetries; attempt++ {
            if attempt > 0 {
                select {
                case <-ctx.Done():
                    return ctx.Err()
                case <-time.After(backoff):
                    backoff *= 2  // 指数退避
                }
            }

            lastErr = invoker(ctx, method, req, reply, cc, opts...)
            if lastErr == nil {
                return nil
            }

            // 只重试可重试的错误码
            code := status.Code(lastErr)
            if code == codes.Unavailable || code == codes.DeadlineExceeded {
                continue
            }
            return lastErr  // 不可重试的错误直接返回
        }
        return lastErr
    }
}
```

---

## 7.4 流拦截器的特殊技巧

### 包装 ClientStream

流拦截器需要返回一个自定义 `ClientStream` 来拦截每个 Send/Recv：

```go
func LoggingStreamInterceptor(ctx context.Context, desc *grpc.StreamDesc,
    cc *grpc.ClientConn, method string, streamer grpc.Streamer,
    opts ...grpc.CallOption) (grpc.ClientStream, error) {

    // 创建底层 stream
    stream, err := streamer(ctx, desc, cc, method, opts...)
    if err != nil {
        return nil, err
    }

    // 返回包装后的 stream
    return &loggingClientStream{ClientStream: stream, method: method}, nil
}

type loggingClientStream struct {
    grpc.ClientStream
    method string
}

func (s *loggingClientStream) SendMsg(m any) error {
    log.Printf("[%s] SendMsg", s.method)
    err := s.ClientStream.SendMsg(m)
    log.Printf("[%s] SendMsg done: %v", s.method, err)
    return err
}

func (s *loggingClientStream) RecvMsg(m any) error {
    log.Printf("[%s] RecvMsg", s.method)
    err := s.ClientStream.RecvMsg(m)
    log.Printf("[%s] RecvMsg done: %v", s.method, err)
    return err
}
```

### 包装 ServerStream

```go
type wrappedServerStream struct {
    grpc.ServerStream
}

func (w *wrappedServerStream) SendMsg(m any) error {
    // 拦截发送
    return w.ServerStream.SendMsg(m)
}

func (w *wrappedServerStream) RecvMsg(m any) error {
    // 拦截接收
    return w.ServerStream.RecvMsg(m)
}
```

---

## 7.5 拦截器与 gRPC 内部机制的交互

### 拦截器在调用链中的位置

```
客户端:
  client.SayHello()
    → ClientConn.Invoke()           # call.go
      → unaryInt() / invoke()       # 拦截器或直接调用
        → newClientStream()         # 创建 Stream
          → Picker.pick()           # 选择后端
          → transport.NewStream()   # HTTP/2 层

服务端:
  http2Server.operateHeaders()
    → Server.handleStream()
      → 路由匹配
        → serviceDesc.Handler()
          → _Greeter_SayHello_Handler()
            → interceptor() / handler()  # 拦截器或直接调用
              → 用户实现的方法
```

### 注意事项

1. **拦截器不影响连接管理**：Resolver、Balancer、Picker 在拦截器之前执行
2. **拦截器不影响重试**：透明重试发生在拦截器之外（由 `newClientStreamWithRetries` 处理）
3. **流拦截器的 CloseSend**：包装的 ClientStream 需要正确代理 `CloseSend()`
4. **Context 传播**：拦截器中修改的 context 会传播到后续拦截器和 handler

---

## 7.6 动手实验：实现链式拦截器

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "runtime/debug"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

// 1. Recovery 拦截器
func recoveryInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = status.Errorf(codes.Internal, "panic: %v", r)
            log.Printf("[RECOVERY] %s: %v\n%s", info.FullMethod, r, debug.Stack())
        }
    }()
    return handler(ctx, req)
}

// 2. 日志拦截器
func loggingInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("[LOG] %s %v code=%s", info.FullMethod, time.Since(start), status.Code(err))
    return resp, err
}

// 3. Auth 拦截器
func authInterceptor(ctx context.Context, req any,
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    // 跳过健康检查
    if info.FullMethod == "/grpc.health.v1.Health/Check" {
        return handler(ctx, req)
    }
    // 验证 token（简化版）
    // ...
    return handler(ctx, req)
}

func main() {
    // 服务端：链式注册
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            recoveryInterceptor,   // 最外层：捕获 panic
            loggingInterceptor,    // 第二层：记录日志
            authInterceptor,       // 最内层：验证认证
        ),
    )
    pb.RegisterGreeterServer(s, &server{})
    go s.Serve(lis)

    // 客户端
    conn, _ := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithChainUnaryInterceptor(
            func(ctx context.Context, method string, req, reply any,
                cc *grpc.ClientConn, invoker grpc.UnaryInvoker,
                opts ...grpc.CallOption) error {
                log.Printf("[CLIENT] calling %s", method)
                err := invoker(ctx, method, req, reply, cc, opts...)
                log.Printf("[CLIENT] %s done: %v", method, status.Code(err))
                return err
            },
        ),
    )
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    resp, err := client.SayHello(context.Background(), &pb.HelloRequest{Name: "world"})
    if err != nil {
        log.Fatalf("error: %v", err)
    }
    fmt.Printf("Response: %s\n", resp.GetMessage())
}
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| 四种拦截器 | 客户端/服务端 × 一元/流式 |
| 洋葱模型 | 前置逻辑 → 下一个拦截器 → 后置逻辑 |
| 链式组装 | 递归构建 invoker 链，最后一个调用真正 RPC |
| 流拦截器 | 包装 ClientStream/ServerStream，拦截 SendMsg/RecvMsg |
| 认证 | 从 metadata 提取 token，验证后注入 context |
| 恢复 | recover() 捕获 panic，转为 gRPC Internal 错误 |
| 限流 | 令牌桶/滑动窗口，超限返回 ResourceExhausted |

---

## 下节预告

第8课将深入 **安全机制：TLS、Token 与认证**，理解 gRPC 的多层安全体系。
