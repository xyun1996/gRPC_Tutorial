# 第5课：负载均衡与服务发现

## 学习目标

- 理解 gRPC 客户端负载均衡架构
- 掌握 Balancer/Picker/SubConn 接口设计
- 追踪 Resolver → Balancer → Picker 的完整协作链路
- 了解 xDS 动态负载均衡

---

## 5.1 负载均衡架构

### 两种负载均衡模式

```
模式1: 代理侧负载均衡（Proxy-side / Server-side）
┌───────┐     ┌──────────┐     ┌───────┐
│Client │────►│  LB Proxy│────►│Backend│
│       │     │ (Nginx/  │     │  1    │
│       │     │  Envoy)  │────►│  2    │
└───────┘     └──────────┘     └───────┘
优点: 客户端简单
缺点: 额外跳转、代理成为瓶颈、非全连接

模式2: 客户端负载均衡（Client-side）── gRPC 原生支持
┌───────┐     ┌───────┐
│Client │────►│Backend│
│  gRPC │────►│  1    │
│       │────►│  2    │
└───────┘     └───────┘
优点: 无额外跳转、低延迟、全连接可见
缺点: 客户端需要感知后端列表
```

gRPC 选择客户端负载均衡，因为微服务间通信需要最低延迟和全连接可见性（用于流控、健康检查等）。

---

## 5.2 Balancer 接口与实现

### 核心接口

`balancer/balancer.go` 定义了负载均衡的核心抽象：

```go
// Builder 创建 Balancer 实例
type Builder interface {
    Build(cc ClientConn, opts BuildOptions) Balancer
    Name() string
}

// Balancer 负载均衡器
type Balancer interface {
    // ★ 接收 Resolver 的新状态（地址列表 + 服务配置）
    UpdateClientConnState(ClientConnState) error

    // Resolver 报告错误
    ResolverError(error)

    // SubConn 状态变化
    UpdateSubConnState(SubConn, SubConnState)

    // 关闭
    Close()
}

// Picker 选择后端 —— 每个 RPC 调用都会执行 Pick
type Picker interface {
    Pick(info PickInfo) (PickResult, error)
}

// ClientConn —— Balancer 通过此接口与 gRPC 框架交互
type ClientConn interface {
    NewSubConn(addrs []resolver.Address, opts NewSubConnOptions) (SubConn, error)
    UpdateAddresses(SubConn, []resolver.Address)
    UpdateState(State)  // ★ 报告新状态 + 新 Picker
    // ...
}

// SubConn 代表一个后端连接
type SubConn interface {
    UpdateAddresses([]resolver.Address)
    Connect()
    Shutdown()
}
```

### Balancer 注册表

```go
// balancer/balancer.go
var m = make(map[string]Builder)

func Register(b Builder) {
    m[b.Name()] = b
}

func Get(name string) Builder {
    return m[name]
}
```

内置 Balancer 通过 `init()` 注册：

```go
// clientconn.go 中的 import
_ "google.golang.org/grpc/balancer/roundrobin"  // 注册 roundrobin

// 默认使用 pick_first
PickFirstBalancerName = pickfirst.Name
```

---

## 5.3 SubConn 状态管理与 Picker 机制

### SubConn 的生命周期

```
                    NewSubConn()
                         │
                         ▼
                    ┌──────────┐
          Connect() │   IDLE    │ ◄── 新创建
                    └────┬─────┘
                         │
                         ▼
                   ┌───────────┐
                   │ CONNECTING │ ◄── 正在建立连接
                   └─────┬─────┘
                         │
               ┌─────────┴─────────┐
               │                   │
               ▼                   ▼
         ┌─────────┐      ┌────────────────┐
         │  READY   │      │TRANSIENTFAILURE│
         │(可发RPC) │      │ (连接失败)      │
         └────┬────┘      └───────┬────────┘
              │                   │
              │ 连接断开           │ 重试
              ▼                   │
         ┌───────────┐            │
         │ CONNECTING │◄───────────┘
         └─────┬─────┘
               │ Shutdown()
               ▼
          ┌──────────┐
          │ SHUTDOWN  │ ◄── 终态
          └──────────┘
```

### Picker 的工作机制

每个 RPC 调用都经过 Picker 选择后端。`picker_wrapper.go` 实现了 Picker 的包装和阻塞逻辑：

```go
// picker_wrapper.go
type pickerGeneration struct {
    picker     balancer.Picker   // 当前的 Picker
    blockingCh chan struct{}     // Picker 失效时关闭，唤醒阻塞的 RPC
}

type pickerWrapper struct {
    pickerGen atomic.Pointer[pickerGeneration]
}

// updatePicker —— Balancer 产生新 Picker 时调用
func (pw *pickerWrapper) updatePicker(p balancer.Picker) {
    old := pw.pickerGen.Swap(&pickerGeneration{
        picker:     p,
        blockingCh: make(chan struct{}),
    })
    close(old.blockingCh)  // 关闭旧 channel，唤醒阻塞的 RPC
}
```

### Pick 的完整逻辑

```go
// picker_wrapper.go 中的 pick 方法（简化版）
func (pw *pickerWrapper) pick(ctx context.Context, ...) (*balancer.PickResult, error) {
    for {
        pg := pw.pickerGen.Load()
        if pg.picker == nil {
            return nil, ErrClientConnClosing
        }

        // 尝试 Pick
        result, err := pg.picker.Pick(info)
        if err == nil {
            return &result, nil  // 成功选中后端
        }

        switch err {
        case balancer.ErrNoSubConnAvailable:
            // 没有 READY 的 SubConn，阻塞等待
            select {
            case <-ctx.Done():
                return nil, ctx.Err()
            case <-pg.blockingCh:
                // 新 Picker 到了，重试
                continue
            }

        case balancer.ErrTransientFailure:
            // 临时失败，返回错误
            return nil, status.Error(codes.Unavailable, err.Error())

        default:
            return nil, err
        }
    }
}
```

---

## 5.4 内置 Balancer 实现

### pick_first（默认）

最简单的策略——选择第一个 READY 的 SubConn：

```go
// balancer/pickfirst/pickfirst.go（简化逻辑）

// 工作方式:
// 1. Resolver 给出 [addr1, addr2, addr3]
// 2. 创建一个 SubConn 包含所有地址
// 3. 按顺序尝试连接: addr1 → addr2 → addr3
// 4. 第一个成功的成为 READY
// 5. 所有 RPC 发往这一个后端

func (p *pickfirstPicker) Pick(info PickInfo) (PickResult, error) {
    if p.subConn != nil {
        return PickResult{SubConn: p.subConn}, nil
    }
    return PickResult{}, ErrNoSubConnAvailable
}
```

### round_robin

轮询分配 RPC 到所有 READY 的 SubConn：

```go
// balancer/roundrobin/roundrobin.go（简化逻辑）

type rrPicker struct {
    subConns []balancer.SubConn
    next     uint32  // 原子递增
}

func (p *rrPicker) Pick(info PickInfo) (PickResult, error) {
    if len(p.subConns) == 0 {
        return PickResult{}, ErrNoSubConnAvailable
    }
    // 原子递增，取模实现轮询
    idx := atomic.AddUint32(&p.next, 1)
    sc := p.subConns[idx%uint32(len(p.subConns))]
    return PickResult{SubConn: sc}, nil
}
```

### 使用 round_robin

```go
conn, err := grpc.NewClient("dns:///example.com:443",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
)
```

---

## 5.5 Resolver → Balancer → Picker 协作链路

### 完整数据流

```
Resolver                ClientConn               Balancer              Picker
  │                        │                        │                    │
  │  DNS查询得到新地址列表    │                        │                    │
  │── UpdateState() ──────►│                        │                    │
  │                        │                        │                    │
  │                  ccResolverWrapper               │                    │
  │                  .UpdateState()                  │                    │
  │                        │                        │                    │
  │                        │ updateResolverStateAndUnlock()              │
  │                        │                        │                    │
  │                        │── UpdateClientConnState() ──►│              │
  │                        │   {Addresses: [...]}    │                    │
  │                        │                        │                    │
  │                        │                 Balancer 处理:               │
  │                        │                 1. 比较 old/new 地址         │
  │                        │                 2. 创建新 SubConn           │
  │                        │                 3. 关闭旧 SubConn           │
  │                        │                 4. 生成新 Picker            │
  │                        │                        │                    │
  │                        │◄── UpdateState() ──────│                    │
  │                        │   {State: READY,       │                    │
  │                        │    Picker: rrPicker}   │                    │
  │                        │                        │                    │
  │                        │ updatePicker()         │                    │
  │                        │────────────────────────│───────────────────►│
  │                        │                        │   新 Picker 生效    │
  │                        │                        │                    │
  │                        │  RPC 到来               │                    │
  │                        │── pick() ──────────────────────────────────►│
  │                        │◄── PickResult{SubConn} ────────────────────│
  │                        │                        │                    │
```

### 源码追踪：从 Resolver 到 Picker

```go
// 1. Resolver 发现新地址
// resolver_wrapper.go
func (ccr *ccResolverWrapper) UpdateState(s resolver.State) error {
    ccr.curState = s
    // 传播到 ClientConn
    ccr.cc.updateResolverStateAndUnlock(s, nil)
    return nil
}

// 2. ClientConn 传递给 Balancer
// clientconn.go
func (cc *ClientConn) updateResolverStateAndUnlock(s resolver.State, err error) {
    // 传递给 Balancer
    cc.balancerWrapper.updateClientConnState(s)
}

// 3. Balancer 处理并生成新 Picker
// balancer_wrapper.go
func (ccb *ccBalancerWrapper) updateClientConnState(s resolver.State) {
    ccb.balancer.UpdateClientConnState(balancer.ClientConnState{
        ResolverState: s,
    })
}

// 4. Balancer 报告新 Picker
// balancer 实现调用 cc.UpdateState()
func (ccb *ccBalancerWrapper) UpdateState(s balancer.State) {
    cc.pickerWrapper.updatePicker(s.Picker)  // 更新 Picker
    cc.csMgr.updateState(s.ConnectivityState)  // 更新连接状态
}
```

---

## 5.6 动手实验：实现自定义 Balancer

### 实验：加权随机 Balancer

```go
package balancer

import (
    "math/rand"
    "sync/atomic"

    "google.golang.org/grpc/balancer"
    "google.golang.org/grpc/balancer/base"
    "google.golang.org/grpc/resolver"
)

const WeightedRandomName = "weighted_random"

func init() {
    // 使用 base 框架注册自定义 Balancer
    balancer.Register(base.NewBalancerBuilder(
        WeightedRandomName,
        &weightedRandomPickerBuilder{},
        base.Config{HealthCheck: true},
    ))
}

// PickerBuilder 创建 Picker
type weightedRandomPickerBuilder struct{}

func (b *weightedRandomPickerBuilder) Build(
    readySCs map[balancer.SubConn]base.SubConnInfo) balancer.Picker {

    var scs []balancer.SubConn
    for sc := range readySCs {
        scs = append(scs, sc)
    }
    return &weightedRandomPicker{subConns: scs}
}

// Picker 实现
type weightedRandomPicker struct {
    subConns []balancer.SubConn
}

func (p *weightedRandomPicker) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
    if len(p.subConns) == 0 {
        return balancer.PickResult{}, balancer.ErrNoSubConnAvailable
    }

    // 随机选择一个后端
    idx := rand.Intn(len(p.subConns))
    return balancer.PickResult{SubConn: p.subConns[idx]}, nil
}
```

### 使用自定义 Balancer

```go
package main

import (
    _ "yourpackage/balancer"  // 注册 weighted_random

    "google.golang.org/grpc"
)

func main() {
    conn, _ := grpc.NewClient("dns:///example.com:50051",
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingConfig": [{"weighted_random":{}}]
        }`),
    )
    defer conn.Close()
    // 所有 RPC 将通过加权随机策略选择后端
}
```

### 实验：观察负载均衡效果

```go
package main

import (
    "context"
    "fmt"
    "log"
    "sync/atomic"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    conn, err := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
    )
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewGreeterClient(conn)

    // 发送 10 个请求，观察分布
    var count atomic.Int32
    for i := 0; i < 10; i++ {
        go func(i int) {
            resp, err := client.SayHello(context.Background(),
                &pb.HelloRequest{Name: fmt.Sprintf("request-%d", i)})
            if err != nil {
                log.Printf("RPC %d failed: %v", i, err)
                return
            }
            n := count.Add(1)
            fmt.Printf("[%d] %s\n", n, resp.GetMessage())
        }(i)
    }

    time.Sleep(5 * time.Second)
    fmt.Printf("Total successful RPCs: %d\n", count.Load())
}
```

> 启动多个 helloworld server 实例（不同端口），观察请求被均匀分配。

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| 客户端 LB | gRPC 原生支持，无代理开销 |
| Builder/Balancer | 工厂模式，Build 创建实例 |
| SubConn | 代表一个后端连接，有独立状态机 |
| Picker | 每次 RPC 选择后端，支持阻塞等待 |
| 协作链路 | Resolver → ClientConn → Balancer → Picker |
| pick_first | 默认策略，选第一个可用后端 |
| round_robin | 轮询分配，需 ServiceConfig 启用 |

---

## 下节预告

第6课将深入 **四种调用模式与流控机制**，理解 Unary/Streaming RPC 的实现差异和 HTTP/2 流量控制。
