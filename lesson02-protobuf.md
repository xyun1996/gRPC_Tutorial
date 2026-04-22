# 第2课：Protocol Buffers 深度解析

## 学习目标

- 理解 .proto 文件到 Go 代码的完整编译链路
- 掌握 Protobuf 编码原理：Varint、TLV、字段布局
- 理解前后兼容性的设计原理
- 解读生成的 Stub 代码结构

---

## 2.1 .proto 文件到 Go 代码的完整编译链路

### 一个简单的 .proto 文件

```protobuf
// helloworld.proto
syntax = "proto3";

package helloworld;

option go_package = "google.golang.org/grpc/examples/helloworld/helloworld";

// 服务定义 —— gRPC 的核心起点
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 消息定义 —— 数据结构
message HelloRequest {
  string name = 1;   // 字段号 = 1
}

message HelloReply {
  string message = 1;
}
```

### 编译链路全景

```
helloworld.proto
       │
       ▼  ┌─────────────────────┐
       │  │   protoc 编译器      │
       │  │   (C++ 实现)        │
       │  └─────────────────────┘
       │         │
       │         ├── protoc-gen-go ──────► helloworld.pb.go
       │         │   (消息类型 + 序列化)    ├── type HelloRequest struct{}
       │         │                         ├── func (x *HelloRequest) Reset()
       │         │                         ├── func (x *HelloRequest) String() string
       │         │                         ├── func (x *HelloRequest) ProtoMessage()
       │         │                         ├── func (m *HelloRequest) GetName() string
       │         │                         └── func init() { proto.RegisterType(...) }
       │         │
       │         └── protoc-gen-go-grpc ──► helloworld_grpc.pb.go
       │             (服务 Stub)            ├── GreeterClient interface
       │                                   ├── greeterClient struct{}
       │                                   ├── GreeterServer interface
       │                                   ├── RegisterGreeterServer()
       │                                   └── UnimplementedGreeterServer struct{}
       ▼
  Go 源码文件
```

### 实际编译命令

```bash
# 安装 protoc 编译器
# https://github.com/protocolbuffers/protobuf/releases

# 安装 Go 插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 编译
protoc --go_out=. --go-grpc_out=. helloworld.proto
```

---

## 2.2 编码原理：Varint、Tag-Length-Value、字段布局

### Varint 编码：变长整数

Protobuf 的基础编码是 **Varint**（可变长度整数），小数字用更少字节：

```
数字          十进制         Varint 字节
1            0x01          [01]                    # 1 字节
300          0x012C        [AC 02]                 # 2 字节
150          0x96          [96 01]                 # 2 字节
1,000,000    0x0F4240      [C0 84 3D]              # 3 字节
```

**Varint 编码规则**：
- 每个字节的最高位（MSB）是"继续位"
  - `1` = 后面还有字节
  - `0` = 这是最后一个字节
- 有效数据使用低 7 位
- 小端序排列

```
编码 300 的过程：

300 的二进制 = 0000 0001 0010 1100

拆成 7 位一组（从低位开始）：
  组1: 010 1100  (低 7 位)
  组2: 000 0010  (高 7 位)

加上继续位：
  组1: 1_0101100 = 0xAC  (还有后续)
  组2: 0_0000010 = 0x02  (结束)

结果: [0xAC, 0x02]
```

### Tag-Length-Value (TLV) 编码

Protobuf 的每个字段编码为 **Tag + [Length] + Value** 格式：

```
┌─────────────┬──────────────┬───────────────┐
│    Tag      │   Length     │    Value       │
│ (字段号+类型)│ (可选，变长)  │  (字段值)      │
└─────────────┴──────────────┴───────────────┘
```

**Tag 的构成**：

```
Tag = (field_number << 3) | wire_type

wire_type（3 bit）:
  0 = Varint         (int32, int64, uint32, uint64, sint32, sint64, bool, enum)
  1 = 64-bit         (fixed64, sfixed64, double)
  2 = Length-delimited (string, bytes, embedded messages, packed repeated)
  5 = 32-bit         (fixed32, sfixed32, float)
```

### 完整编码示例

```protobuf
message HelloRequest {
  string name = 1;   // field_number=1, wire_type=2 (Length-delimited)
}
```

编码 `name = "world"` 的过程：

```
1. 计算 Tag:
   field_number = 1, wire_type = 2
   Tag = (1 << 3) | 2 = 0x0A

2. 编码 Length:
   "world" 长度 = 5 字节
   Length = 0x05 (Varint)

3. 编码 Value:
   "world" 的 UTF-8 字节 = 77 6F 72 6C 64

完整编码结果: 0A 05 77 6F 72 6C 64
             │  │  └──────────────┘
             │  │   "world" (5字节)
             │  └── Length = 5
             └───── Tag = 0x0A (field 1, length-delimited)
```

### 多字段消息编码

```protobuf
message Person {
  string name = 1;    // Tag = 0x0A
  int32  id   = 2;    // Tag = 0x10 (field 2, varint)
  string email = 3;   // Tag = 0x1A
}
```

编码 `Person{name: "Alice", id: 42, email: "a@b.com"}`：

```
0A 05 41 6C 69 63 65     # name = "Alice"
10 2A                     # id = 42 (0x2A)
1A 05 61 40 62 2E 63 6F 6D  # email = "a@b.com"
```

**关键特性**：
- 字段按出现顺序编码（proto3 无 required，未设置的字段不编码）
- 解码器根据 Tag 识别字段，**字段顺序无关紧要**
- 缺失字段使用默认值（string="", int=0）

### ZigZag 编码：负数优化

普通 Varint 对负数效率极差（负数被当作超大无符号数），ZigZag 编码解决此问题：

```
原始值          ZigZag值        Varint字节
 0              0               [00]
-1              1               [01]
 1              2               [02]
-2              3               [03]
 2              4               [04]
-3              5               [05]

公式: ZigZag(n) = (n << 1) ^ (n >> 31)    (int32)
     ZigZag(n) = (n << 1) ^ (n >> 63)    (int64)
```

---

## 2.3 前后兼容性设计

### 兼容性规则

Protobuf 设计的核心目标是**前后兼容**——新代码能读旧数据，旧代码能读新数据。

```
┌───────────────────────────────┬────────────────────────────┐
│         安全的操作             │       不安全的操作          │
├───────────────────────────────┼────────────────────────────┤
│ 添加新字段（使用新字段号）      │ 修改已有字段的类型          │
│ 删除字段（保留字段号，不重用）  │ 重用已删除的字段号          │
│ 添加新的 oneof 成员           │ 删除后重入 oneof 成员       │
│ required → optional (proto2) │ 修改字段号                 │
│ 重命名字段（二进制不变）       │ 修改 wire type             │
└───────────────────────────────┴────────────────────────────┘
```

### 为什么字段号不能重用？

```
版本1: message Foo { string name = 1; }
版本2: message Foo { int64 age = 1; }      // 危险！重用了字段号 1

旧数据编码: 0A 05 ... (Tag 0x0A = field 1, length-delimited)
新代码解码: Tag 0x0A → field 1 → int64 → 尝试按 Varint 解码字符串！
结果: 数据损坏，无报错
```

### 保留字段号的约定

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;      // 保留字段号，禁止使用
  reserved "foo", "bar";         // 保留字段名，禁止使用
}
```

### Repeated 字段的两种编码

```
非 packed (proto2 默认):
  field 1 = [10, 20, 30]
  → Tag1 Len1 Value1 | Tag1 Len1 Value2 | Tag1 Len1 Value3

Packed (proto3 默认):
  field 1 = [10, 20, 30]
  → Tag1 TotalLen Value1 Value2 Value3
  → 0A 03 0A 14 1E       # 3 字节打包
```

---

## 2.4 生成的 Stub 代码结构解读

### helloworld.pb.go —— 消息类型

```go
// 生成的 HelloRequest 结构体
type HelloRequest struct {
    state         protoimpl.MessageState  // 内部状态
    sizeCache     protoimpl.SizeCache      // 序列化大小缓存
    unknownFields protoimpl.UnknownFields   // 未知字段保留

    Name string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
}

// Getter 方法
func (x *HelloRequest) GetName() string {
    if x != nil {
        return x.Name
    }
    return ""
}

// ProtoMessage 标记接口
func (*HelloRequest) ProtoMessage() {}

// Reset 重置为默认值
func (x *HelloRequest) Reset() { *x = HelloRequest{} }

// String 返回字符串表示
func (x *HelloRequest) String() string { return proto.CompactTextString(x) }
```

**关键结构标签解析**：
```go
Name string `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty"`
                     │     │  │       │      │
                     │     │  │       │      └── proto3 语法
                     │     │  │       └── 原始字段名
                     │     │  └── opt (optional，proto3 全是 optional)
                     │     └── 字段号 = 1
                     └── wire type: bytes = length-delimited
```

### helloworld_grpc.pb.go —— 服务 Stub

**客户端接口**：
```go
// 客户端接口 —— 用户面向此接口编程
type GreeterClient interface {
    SayHello(ctx context.Context, in *HelloRequest,
        opts ...grpc.CallOption) (*HelloReply, error)
}

// 客户端实现 —— 由 grpc.ClientConn 驱动
type greeterClient struct {
    cc grpc.ClientConnInterface   // 持有 ClientConn 引用
}

func newGreeterClient(cc grpc.ClientConnInterface) GreeterClient {
    return &greeterClient{cc}
}

// ★ 核心方法：调用链路的入口
func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest,
    opts ...grpc.CallOption) (*HelloReply, error) {

    out := new(HelloReply)
    // 关键：调用 ClientConn.Invoke()，这是 gRPC 框架的入口
    err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
    if err != nil {
        return nil, err
    }
    return out, nil
}
```

**注意 `/helloworld.Greeter/SayHello` 这个路径**——这正是 gRPC 在 HTTP/2 HEADERS 帧中 `:path` 头的值。

**服务端接口**：
```go
// 服务端接口 —— 用户需要实现此接口
type GreeterServer interface {
    SayHello(context.Context, *HelloRequest) (*HelloReply, error)
    mustEmbedUnimplementedGreeterServer()  // 编译期安全检查
}

// 服务描述符 —— 注册服务时使用
var Greeter_ServiceDesc = grpc.ServiceDesc{
    ServiceName: "helloworld.Greeter",
    HandlerType: (*GreeterServer)(nil),
    Methods: []grpc.MethodDesc{{
        MethodName: "SayHello",
        Handler:    _Greeter_SayHello_Handler,  // 内部分发函数
    }},
    Streams:  []grpc.StreamDesc{},
    Metadata: "helloworld.proto",
}

// 注册函数
func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
    s.RegisterService(&Greeter_ServiceDesc, srv)
}

// 内部分发函数：从请求体解码 → 调用用户 Handler → 编码响应
func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context,
    dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {

    in := new(HelloRequest)
    if err := dec(in); err != nil {  // 反序列化请求
        return nil, err
    }
    if interceptor == nil {
        return srv.(GreeterServer).SayHello(ctx, in)  // 无拦截器，直接调用
    }
    info := &grpc.UnaryServerInfo{
        Server:     srv,
        FullMethod: "/helloworld.Greeter/SayHello",
    }
    handler := func(ctx context.Context, req interface{}) (interface{}, error) {
        return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
    }
    return interceptor(ctx, in, info, handler)  // 经过拦截器
}
```

### gRPC 代码生成的四种模式

```
.proto 定义                          生成的客户端方法签名
─────────────────────────────────────────────────────────────
rpc SayHello(Req) returns (Resp)     SayHello(ctx, *Req) (*Resp, error)
rpc Stream(Req) returns (stream R)   Stream(ctx, *Req) (Greeter_StreamClient, error)
rpc Upload(stream Req) returns (R)   Upload(ctx) (Greeter_UploadClient, error)
rpc Chat(stream Req) returns (str R) Chat(ctx) (Greeter_ChatClient, error)
```

---

## 2.5 动手实验：手写二进制解码

### 实验1：编码并观察 Protobuf 二进制

```go
package main

import (
    "fmt"

    pb "google.golang.org/grpc/examples/helloworld/helloworld"
    "google.golang.org/protobuf/proto"
)

func main() {
    // 创建消息
    req := &pb.HelloRequest{Name: "world"}

    // 序列化
    data, err := proto.Marshal(req)
    if err != nil {
        panic(err)
    }

    // 打印原始字节
    fmt.Printf("Encoded bytes (%d): ", len(data))
    for _, b := range data {
        fmt.Printf("%02X ", b)
    }
    fmt.Println()

    // 输出: 0A 05 77 6F 72 6C 64
    //        │  │  └──────────────┘
    //        │  │   "world" (77 6F 72 6C 64)
    //        │  └── Length = 5
    //        └───── Tag = 0x0A (field 1, length-delimited)
}
```

### 实验2：手写 Varint 解码器

```go
package main

import "fmt"

// 手写 Varint 解码
func decodeVarint(data []byte) (value uint64, bytesRead int) {
    var result uint64
    var shift uint
    for i, b := range data {
        result |= uint64(b&0x7F) << shift  // 取低 7 位
        if b&0x80 == 0 {                     // 检查继续位
            return result, i + 1
        }
        shift += 7
    }
    return result, len(data)
}

// 手写 Tag 解码
func decodeTag(tagBytes []byte) (fieldNumber int, wireType int, bytesRead int) {
    value, n := decodeVarint(tagBytes)
    return int(value >> 3), int(value & 0x7), n
}

func main() {
    // 解码 "HelloRequest{name=world}" 的编码
    data := []byte{0x0A, 0x05, 0x77, 0x6F, 0x72, 0x6C, 0x64}

    offset := 0

    // 解码 Tag
    fieldNum, wireType, n := decodeTag(data[offset:])
    offset += n
    fmt.Printf("Tag: field_number=%d, wire_type=%d\n", fieldNum, wireType)
    // 输出: Tag: field_number=1, wire_type=2

    // wire_type=2 是 Length-delimited，需要解码 Length
    length, n := decodeVarint(data[offset:])
    offset += n
    fmt.Printf("Length: %d\n", length)
    // 输出: Length: 5

    // 解码 Value
    value := string(data[offset : offset+int(length)])
    fmt.Printf("Value: %s\n", value)
    // 输出: Value: world
}
```

### 实验3：观察嵌套消息的编码

```protobuf
message Outer {
  Inner inner = 1;
  int32 count = 2;
}

message Inner {
  string value = 1;
}
```

```go
func main() {
    outer := &Outer{
        Inner: &Inner{Value: "hello"},
        Count: 42,
    }
    data, _ := proto.Marshal(outer)

    // 外层消息：
    // Tag(field=1, type=length-delimited) + Length + [内层消息编码]
    // Tag(field=2, type=varint) + Value(42)
    //
    // 0A 07 0A 05 68 65 6C 6C 6F 10 2A
    // │  │  │  │  └───────────────┘  │  │
    // │  │  │  │   "hello"           │  └── count=42
    // │  │  │  └── Inner Length=5    └── Tag(field=2, varint)
    // │  │  └── Inner Tag(field=1, length-delimited)
    // │  └── Outer Length=7 (内层消息总长度)
    // └── Outer Tag(field=1, length-delimited)
}
```

**嵌套消息的精妙之处**：内层消息被当作 `bytes` 编码在外层中，实现了递归编码——解码器可以逐层剥开。

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| 编译链路 | .proto → protoc → .pb.go(消息) + _grpc.pb.go(Stub) |
| Varint | 变长整数编码，小数字省空间，MSB 为继续位 |
| TLV | Tag=(field_number<<3)\|wire_type，标识字段号和类型 |
| 兼容性 | 字段号不可重用，可安全添加字段，不可修改类型 |
| Stub 代码 | 客户端调 `cc.Invoke()`，服务端通过 `ServiceDesc` 注册 |

---

## 下节预告

第3课将深入 **HTTP/2 与 gRPC 帧协议**，理解 Varint/TLV 编码后的数据如何被封装到 HTTP/2 帧中在网络传输。
