# 第8课：安全机制：TLS、Token 与认证

## 学习目标

- 理解 TLS 握手与 ALPN 协商 HTTP/2 的过程
- 掌握单向认证与双向认证（mTLS）的区别与配置
- 理解 PerRPCCredentials 与 ChannelCredentials 的组合
- 追踪 credentials 包的源码实现

---

## 8.1 TLS 握手与 ALPN 协商 HTTP/2

### TLS 在 gRPC 中的角色

gRPC 强制要求传输层安全（除非显式使用 `insecure.NewCredentials()`）。TLS 承担三个职责：

```
1. 加密: 防止窃听
2. 完整性: 防止篡改
3. 认证: 通过证书验证身份
```

### ALPN 与 HTTP/2 协商

HTTP/2 需要通过 TLS 的 ALPN（Application-Layer Protocol Negotiation）来协商：

```
TLS 握手过程中的 ALPN 协商:

客户端                              服务端
  │── ClientHello ──────────────────►│
  │   ALPN: [h2, http/1.1]          │  ← 客户端声明支持 h2
  │                                  │
  │◄── ServerHello ─────────────────│
  │   ALPN: h2                      │  ← 服务端选择 h2
  │                                  │
  │◄── Certificate ────────────────│
  │── Key Exchange ────────────────►│
  │◄── Finished ───────────────────│
  │── Finished ─────────────────────►│
  │                                  │
  │     TLS 握手完成，开始 HTTP/2     │
```

**关键点**：如果 ALPN 协商失败（服务端不支持 h2），gRPC 连接将失败。

### 源码中的 TLS 握手

`internal/transport/http2_client.go`：

```go
func newHTTP2Client(...) (*http2Client, error) {
    // 1. TCP 连接
    conn, err := dial(connectCtx, opts.Dialer, addr, ...)

    // 2. TLS 握手
    if creds := opts.TransportCredentials; creds != nil {
        conn, authInfo, err = creds.ClientHandshake(
            connectCtx, opts.BaseConfig.Authority, conn)
    }

    // 3. TLS 握手成功后，开始 HTTP/2 通信
    framer := newFramer(conn, ...)
    // ...
}
```

---

## 8.2 证书体系：单向认证 vs 双向认证（mTLS）

### 单向认证（Server Authentication）

最常见的模式——客户端验证服务端身份：

```
客户端                              服务端
  │── ClientHello ──────────────────►│
  │◄── ServerHello + Certificate ───│  服务端发送证书
  │                                  │
  │  客户端验证:                      │
  │  1. 证书由受信任 CA 签发？         │
  │  2. 证书未过期？                  │
  │  3. 证书的 CN/SAN 匹配目标主机？  │
  │                                  │
  │── Key Exchange ────────────────►│
  │◄── Finished ───────────────────│
  │     通信加密，但服务端不验证客户端   │
```

```go
// 客户端配置
conn, err := grpc.NewClient("example.com:443",
    grpc.WithTransportCredentials(
        credentials.NewTLS(&tls.Config{
            // 默认使用系统 CA 证书池
            // ServerName 默认使用目标主机名
        }),
    ),
)
```

### 双向认证（Mutual TLS / mTLS）

服务端也验证客户端身份——用于服务间通信：

```
客户端                              服务端
  │── ClientHello + Certificate ────►│  客户端也发送证书
  │◄── ServerHello + Certificate ───│
  │                                  │
  │  双方互相验证:                     │
  │  客户端验证服务端证书               │
  │  服务端验证客户端证书               │
  │                                  │
  │── Key Exchange ────────────────►│
  │◄── Finished ───────────────────│
  │     双向认证完成，双方身份已确认     │
```

```go
// 服务端配置 mTLS
creds, err := credentials.NewServerTLSFromFile(
    "server.crt", "server.key",   // 服务端证书和私钥
)

s := grpc.NewServer(grpc.Creds(creds))

// 客户端配置 mTLS
cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
caCert, _ := os.ReadFile("ca.crt")
caPool := x509.NewCertPool()
caPool.AppendCertsFromPEM(caCert)

conn, err := grpc.NewClient("example.com:443",
    grpc.WithTransportCredentials(
        credentials.NewTLS(&tls.Config{
            Certificates: []tls.Certificate{cert},  // 客户端证书
            RootCAs:      caPool,                    // 信任的 CA
        }),
    ),
)
```

---

## 8.3 Token 认证：Per-RPC Credentials

### 两种凭证类型

```
┌──────────────────────────┬──────────────────────────────────┐
│    ChannelCredentials    │     PerRPCCredentials             │
├──────────────────────────┼──────────────────────────────────┤
│ 连接级别                  │ 每次 RPC 级别                     │
│ TLS 证书                 │ OAuth2 / JWT / 自定义 Token       │
│ 在连接建立时协商           │ 在每个 RPC 的 metadata 中携带     │
│ NewTLS() / NewClientTLS │ oauth.NewOauthAccess()            │
└──────────────────────────┴──────────────────────────────────┘
```

### PerRPCCredentials 接口

`credentials/credentials.go`：

```go
type PerRPCCredentials interface {
    // GetRequestMetadata 每次调用 RPC 前被调用
    // 返回的 map 会被添加到 HTTP/2 HEADERS 帧中
    GetRequestMetadata(ctx context.Context, uri ...string) (
        map[string]string, error)

    // RequireTransportSecurity 是否要求传输安全
    // 返回 true 时，在 insecure 连接上使用会报错
    RequireTransportSecurity() bool
}
```

### OAuth2 Token 示例

```go
import "google.golang.org/grpc/credentials/oauth"

// 方式1: 使用 Access Token
conn, _ := grpc.NewClient("example.com:443",
    grpc.WithPerRPCCredentials(
        oauth.NewOauthAccess(&oauth2.Token{
            AccessToken: "ya29.a0AfH6...",
            TokenType:   "Bearer",
        }),
    ),
)

// 方式2: 使用 JWT
conn, _ := grpc.NewClient("example.com:443",
    grpc.WithPerRPCCredentials(
        oauth.NewJWTAccessFromFile("service-account.json"),
    ),
)
```

### Token 注入 HTTP/2 头部

PerRPCCredentials 的 `GetRequestMetadata` 返回的键值对被添加到 HTTP/2 HEADERS：

```
HEADERS 帧:
  :method: POST
  :path: /package.service/method
  authorization: Bearer ya29.a0AfH6...     ← Token 在这里
  content-type: application/grpc
```

---

## 8.4 CallCredentials 与 ChannelCredentials 的组合

### 组合机制

gRPC 允许将 Channel 级凭证（TLS）和 RPC 级凭证（Token）组合：

```go
import "google.golang.org/grpc/credentials"

// 1. 创建 Channel 级凭证（TLS）
transportCreds := credentials.NewTLS(&tls.Config{})

// 2. 创建 RPC 级凭证（Token）
perRPCCreds := oauth.NewOauthAccess(token)

// 3. 组合
combinedCreds := credentials.NewComposedCredentials(transportCreds, perRPCCreds)

// 4. 使用
conn, _ := grpc.NewClient("example.com:443",
    grpc.WithTransportCredentials(combinedCreds),
)
```

### 源码追踪：凭证验证链路

```go
// clientconn.go — NewClient 中的凭证验证
func (cc *ClientConn) validateTransportCredentials() error {
    if cc.dopts.copts.TransportCredentials == nil &&
        cc.dopts.copts.CredsBundle == nil {
        // 没有设置任何凭证 → 报错
        // 除非显式使用 insecure
        return errNoTransportSecurity
    }
    return nil
}

// internal/transport/http2_client.go — 连接建立时使用凭证
func newHTTP2Client(...) {
    // TLS 握手
    conn, authInfo, err = opts.TransportCredentials.ClientHandshake(...)

    // PerRPC 凭证在每次 RPC 时注入 metadata
    // 见 stream.go 中的 newClientStream()
}
```

### 安全级别（SecurityLevel）

`credentials/credentials.go` 定义了三个安全级别：

```go
const (
    NoSecurity         SecurityLevel = iota + 1  // 无安全（insecure）
    IntegrityOnly                                // 仅完整性
    PrivacyAndIntegrity                           // 隐私+完整性（TLS）
)
```

`PerRPCCredentials.RequireTransportSecurity()` 返回 `true` 时，gRPC 会检查安全级别——在 `NoSecurity` 连接上使用会报错。

---

## 8.5 内置凭证实现

```
credentials/           类型             说明
───────────────────────────────────────────────────────────────
tls.go                TransportCreds   标准 TLS 凭证
insecure/             TransportCreds   明文传输（开发用）
oauth/                PerRPCCreds      OAuth2 Access Token
jwt/                  PerRPCCreds      JWT Token（从文件或自定义源）
alts/                 TransportCreds   Google ALTS（应用层传输安全）
google/               CredsBundle      自动选择 ALTS 或 TLS
local/                TransportCreds   本地安全（Unix socket / loopback）
sts/                  PerRPCCreds      Security Token Service 交换
xds/                  CredsBundle      xDS 动态证书供应
credentials/tls/      TransportCreds   高级 TLS（动态证书轮换）
```

---

## 8.6 动手实验：配置 mTLS 双向认证

### 实验：生成自签证书并部署 mTLS

```bash
# 1. 生成 CA 密钥和证书
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt \
    -subj "/CN=MyCA"

# 2. 生成服务端密钥和 CSR
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr \
    -subj "/CN=localhost"

# 3. 用 CA 签发服务端证书
openssl x509 -req -days 365 -in server.csr -CA ca.crt \
    -CAkey ca.key -set_serial 01 -out server.crt

# 4. 生成客户端密钥和 CSR
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr \
    -subj "/CN=client"

# 5. 用 CA 签发客户端证书
openssl x509 -req -days 365 -in client.csr -CA ca.crt \
    -CAkey ca.key -set_serial 02 -out client.crt
```

### mTLS 服务端

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "log"
    "net"
    "os"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    // 加载服务端证书
    cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")

    // 配置 TLS：要求客户端证书
    caCert, _ := os.ReadFile("ca.crt")
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert,  // ★ 要求客户端证书
        ClientCAs:    caPool,                           // ★ 信任的 CA
    }

    creds := credentials.NewTLS(tlsConfig)

    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer(grpc.Creds(creds))
    pb.RegisterGreeterServer(s, &server{})

    log.Println("mTLS server listening on :50051")
    s.Serve(lis)
}
```

### mTLS 客户端

```go
package main

import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "log"
    "os"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

func main() {
    // 加载客户端证书
    cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")

    // 配置 TLS
    caCert, _ := os.ReadFile("ca.crt")
    caPool := x509.NewCertPool()
    caPool.AppendCertsFromPEM(caCert)

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{cert},  // ★ 客户端证书
        RootCAs:      caPool,                    // 验证服务端证书
    }

    creds := credentials.NewTLS(tlsConfig)

    conn, _ := grpc.NewClient("localhost:50051",
        grpc.WithTransportCredentials(creds))
    defer conn.Close()

    client := pb.NewGreeterClient(conn)
    resp, err := client.SayHello(context.Background(),
        &pb.HelloRequest{Name: "mTLS world"})
    if err != nil {
        log.Fatalf("could not greet: %v", err)
    }
    fmt.Printf("Greeting: %s\n", resp.GetMessage())
}
```

### 验证：使用 Peer 获取认证信息

```go
// 在服务端 Handler 中获取客户端身份
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    // 获取对端信息
    peer, ok := peer.FromContext(ctx)
    if ok {
        fmt.Printf("Peer: %s\n", peer.Addr.String())
        if tlsInfo, ok := peer.AuthInfo.(credentials.TLSInfo); ok {
            fmt.Printf("Client CN: %s\n",
                tlsInfo.State.PeerCertificates[0].Subject.CommonName)
        }
    }

    return &pb.HelloReply{Message: "Hello " + req.Name}, nil
}
```

---

## 本课小结

| 概念 | 核心要点 |
|------|---------|
| TLS + ALPN | gRPC 通过 ALPN 协商 h2，TLS 提供加密+认证 |
| 单向认证 | 客户端验证服务端证书（最常见） |
| mTLS | 双向验证，服务端也验证客户端证书 |
| TransportCredentials | 连接级凭证，TLS 握手时使用 |
| PerRPCCredentials | RPC 级凭证，每次调用注入 metadata |
| 组合凭证 | NewComposedCredentials 将两种凭证组合 |
| SecurityLevel | NoSecurity / IntegrityOnly / PrivacyAndIntegrity |

---

## 下节预告

第9课将深入 **可靠性机制：重试、超时与死线传播**，理解 gRPC 如何处理网络故障。
