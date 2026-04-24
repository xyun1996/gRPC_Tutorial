# 附录A：Stats Handler — 可观测性接口

## stats.Handler 接口

`stats.Handler` 是 gRPC 内置的可观测性钩子接口，位于 `stats/stats.go`。它与拦截器不同——拦截器可以修改请求/响应，Stats Handler 只能观察，不能干预。

### 接口定义

```go
type Handler interface {
    TagRPC(context.Context, *RPCTagInfo) context.Context
    HandleRPC(context.Context, RPCStats)

    TagConn(context.Context, *ConnTagInfo) context.Context
    HandleConn(context.Context, ConnStats)
}
```

### Tag 与 Handle 的分工

| 方法 | 角色 | 调用时机 | context 流向 |
|------|------|---------|-------------|
| **TagRPC** | "贴标签" | RPC **开始前** 调用一次 | **返回的 ctx → 后续所有 HandleRPC 的入参** |
| **HandleRPC** | "处理事件" | RPC **过程中** 多次调用 | 接收 TagRPC 返回的 ctx |
| **TagConn** | "贴标签" | 连接**建立前** 调用一次 | **返回的 ctx → 后续 HandleConn / HandleRPC 的入参** |
| **HandleConn** | "处理事件" | 连接**生命周期** 调用 | 接收 TagConn 返回的 ctx |

**Tag 的返回值是数据传递的关键**——你在 TagRPC 中往 context 里存的任何东西，都会出现在后续所有 HandleRPC 回调的 ctx 参数里。

---

## 示例1：最简日志 Handler

```go
type loggingHandler struct{}

func (h *loggingHandler) TagConn(ctx context.Context, info *stats.ConnTagInfo) context.Context {
    log.Printf("[CONN] new connection, client=%v", info.RemoteAddr)
    return ctx
}

func (h *loggingHandler) HandleConn(ctx context.Context, s stats.ConnStats) {
    switch s.(type) {
    case *stats.ConnBegin:
        log.Printf("[CONN] begin")
    case *stats.ConnEnd:
        log.Printf("[CONN] end")
    }
}

func (h *loggingHandler) TagRPC(ctx context.Context, info *stats.RPCTagInfo) context.Context {
    log.Printf("[RPC] %s starting", info.FullMethodName)
    return context.WithValue(ctx, "method", info.FullMethodName)
}

func (h *loggingHandler) HandleRPC(ctx context.Context, s stats.RPCStats) {
    method := ctx.Value("method").(string)
    switch ev := s.(type) {
    case *stats.OutHeader:
        log.Printf("[RPC] %s -> header sent", method)
    case *stats.InHeader:
        log.Printf("[RPC] %s <- header received", method)
    case *stats.OutPayload:
        log.Printf("[RPC] %s -> wire_bytes=%d", method, ev.WireLength)
    case *stats.InPayload:
        log.Printf("[RPC] %s <- wire_bytes=%d", method, ev.WireLength)
    case *stats.End:
        log.Printf("[RPC] %s finished: %v", method, ev.Error)
    }
}
```

使用：

```go
conn, _ := grpc.Dial("localhost:50051",
    grpc.WithStatsHandler(&loggingHandler{}),
)
```

---

## 示例2：OpenTelemetry Tracing（真实案例）

`stats/opentelemetry/client_tracing.go` 是 gRPC 内置的 OTel 客户端 tracing 实现：

```go
// TagRPC: 每次 RPC 开始时创建 OpenTelemetry span，注入 W3C TraceContext 到 metadata
func (h *clientTracingHandler) TagRPC(ctx context.Context, info *stats.RPCTagInfo) context.Context {
    ctx, ai := getOrCreateRPCAttemptInfo(ctx)
    ctx, ai = h.traceTagRPC(ctx, ai, info.NameResolutionDelay)
    // traceTagRPC 内部:
    //   1. tracer.Start(ctx, "Attempt.Send.XXX") → 创建 span
    //   2. TextMapPropagator.Inject() → 将 trace-id 注入 outgoing metadata
    return setRPCInfo(ctx, &rpcInfo{ai: ai})
}

// HandleRPC: 处理每个 RPC 事件，补充 opentelemetry span 信息
func (h *clientTracingHandler) HandleRPC(ctx context.Context, rs stats.RPCStats) {
    ri := getRPCInfo(ctx)
    populateSpan(rs, ri.ai)
}

// TagConn: tracing 不关心连接层面
func (h *clientTracingHandler) TagConn(ctx context.Context, _ *stats.ConnTagInfo) context.Context {
    return ctx
}

// HandleConn: tracing 不关心连接层面
func (h *clientTracingHandler) HandleConn(context.Context, stats.ConnStats) {}
```

---

## RPCStats 事件类型全览

```go
// 所有 RPCStats 实现类型:
*stats.OutHeader      // 请求头已发出
*stats.InHeader       // 收到响应头
*stats.OutPayload     // 消息已编码并发出（含 WireLength、Length 等）
*stats.InPayload      // 收到消息（含 WireLength、Length 等）
*stats.OutTrailer     // 请求 trailer 已发出
*stats.InTrailer      // 收到响应 trailer
*stats.End            // RPC 结束（含最终的 error）
```

## 一次 Unary RPC 的事件时序

```
TagRPC()
  ├─ HandleRPC(OutHeader)    ← HEADERS 帧发出
  ├─ HandleRPC(OutPayload)   ← DATA 帧发出
  ├─ HandleRPC(InHeader)     ← HEADERS 帧收到
  ├─ HandleRPC(InPayload)    ← DATA 帧收到
  └─ HandleRPC(End)          ← 流关闭
```

## ConnStats 事件

```go
// ConnStats 的实现类型:
*stats.ConnBegin   // 连接建立
*stats.ConnEnd     // 连接关闭
```

---

## Stats Handler 与 Interceptor 的区别

| 维度 | Stats Handler | Interceptor |
|------|-------------|-------------|
| 能否修改请求/响应 | 不能 | 能 |
| 能否中止 RPC | 不能 | 能 |
| 注册方式 | `WithStatsHandler()` | 链式注册 |
| 典型用途 | 监控、追踪、日志 | 认证、限流、重试、业务逻辑 |
