# 附录B：关于废弃代码的 FAQ

## 为什么 grpc-go 里有这么多 Deprecated 代码？

截至当前版本（v1.82.0-dev），代码库中有约 370 项 `Deprecated` 标注。这不是代码腐烂，而是有意为之。

### 主要原因

**1. 向后兼容承诺**

gRPC 遵循 semantic versioning，v1.x 大版本内不能删除公开 API。只能标记 Deprecated，等 v2 才能移除。gRPC 的 deprecation 周期特别长，因为：

- Google 内部有海量 gRPC 调用，自身代码也需要迁移时间
- 社区有数万个自定义 Balancer/Resolver 实现依赖旧接口
- 典型周期：deprecate → 几个 minor 版本 → 再几个版本后才删除

**2. 大部分是 protobuf 自动生成的**

370 条 Deprecated 中，绝大部分是 protobuf 自动生成的 `ProtoReflect.Descriptor` 之类，属于编程语言无关的通告机制，不是 Go 特有的问题。

**3. Balancer API 演进**

这是最有代表性的 deprecation 案例：

```
旧版 Balancer API (balancer.go):
  ├─ NewSubConn([]Address)       → 废弃: 每个 SubConn 应只对应一个地址
  ├─ RemoveSubConn               → 废弃: 改用 SubConn.Shutdown
  ├─ UpdateAddresses             → 废弃: 改用新建 SubConn
  └─ UpdateSubConnState          → 废弃: 改用 NewSubConnOptions.StateListener
```

同一个重构涉及了 4 个 API 的废弃，新旧接口同时存在以保持兼容。

### 学习建议

- **学 API 时看新不看旧** — `Deprecated` 注释里有 `Use XXX instead`
- **读流程时新旧都要认** — 内部实现（如 `acBalancerWrapper`）同时实现了新旧两套接口
- **写新代码时永远用新 API**

### gRPC 还有人用吗？

**是云原生微服务通信的事实标准。**

- 当前活跃版本 v1.82.0-dev，每天有 commit
- Kubernetes、etcd、Istio、Envoy 全部依赖 gRPC
- 所有主流编程语言都有官方实现（Go、Java、C++、Python、Node、Rust 等）

废弃代码多恰恰说明 **用的人多** — 不敢随便删，只能温和引导升级。一个没人用的项目根本不需要 deprecate。
