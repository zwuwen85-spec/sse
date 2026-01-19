# gRPC微服务架构设计文档

> 项目：GoDemo 微服务系统
> 创建时间：2025-01-16
> 作者：Claude Code

---

## 目录

1. [架构概述](#1-架构概述)
2. [通信方式选择](#2-通信方式选择)
3. [内部gRPC调用](#3-内部grpc调用)
4. [gRPC高并发支持](#4-grpc高并发支持)
5. [部署架构](#5-部署架构)
6. [代码示例](#6-代码示例)

---

## 1. 架构概述

### 1.1 项目模块

本项目采用微服务架构，包含以下核心模块：

| 模块 | HTTP端口 | gRPC端口 | 职责 |
|------|----------|----------|------|
| gow.home.busi | 19001 | - | IM业务服务（主应用） |
| gow.user.base | 17001 | - | 用户中心服务 |
| gow.live.base | 17005 | - | 直播服务 |
| gow.balance.base | 17006 | 17007 | 余额/支付服务 |
| gow.gift.base | 17008 | 17007 | 礼物服务 |
| gow.sse.base | - | gRPC | 推送服务 |
| gow.common.lib | - | - | 公共库/共享组件 |

### 1.2 架构原则

**对外用 HTTP，对内用 gRPC**

```
┌─────────────────────────────────────────────────────────────┐
│                      客户端层                                 │
│   (移动端 App / Web 前端 / 第三方服务)                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    HTTP/JSON 请求
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Nginx Ingress 网关                         │
│              test-api.playpop.cc                            │
│  根据路径前缀路由到不同服务                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
        ┌───────────────────┴───────────────────┐
        ↓                   ↓                   ↓
  /woolive          /woolive/balanceservice  /woolive/giftservice
        ↓                   ↓                   ↓
┌──────────┐    ┌──────────────┐      ┌──────────────┐
│home.busi │    │balance.base  │      │gift.base     │
│(主业务)  │◄───┤(余额服务)    │◄─────┤(礼物服务)    │
└──────────┘    └──────────────┘      └──────────────┘
                      │
                      │ 内部调用: gRPC (Protobuf)
                      │
                      ↓
              其他微服务模块
```

---

## 2. 通信方式选择

### 2.1 HTTP 用于

| 场景 | 说明 | 示例 |
|------|------|------|
| 客户端直接访问 | 移动端/Web端需要RESTful API | 用户信息查询 |
| 第三方对接 | JSON格式通用、易调试 | 开放平台API |
| 管理后台 | 兼容性强，可用浏览器测试 | 后台管理系统 |
| 低频操作 | 实时性要求不高的业务 | 配置获取 |

### 2.2 gRPC 用于

| 场景 | 说明 | 示例 |
|------|------|------|
| 内部服务通信 | Go模块间高性能调用 | home → balance |
| 高频交易 | 实时性要求高的业务 | 金币冻结/解冻 |
| 性能敏感 | 需要低延迟的场景 | 礼物赠送实时处理 |
| 严格契约 | 需要类型安全的接口 | 金融交易 |

### 2.3 性能对比

| 维度 | HTTP/JSON | gRPC |
|------|-----------|------|
| 序列化 | 文本JSON，较慢 | 二进制Protobuf，快3-5倍 |
| 连接复用 | HTTP/1.1 每请求独立连接 | HTTP/2 多路复用 |
| 类型安全 | 运行时检查 | 编译时检查 |
| 可调试性 | 浏览器直接测试 | 需要工具 |
| 契约定义 | 无统一标准 | Proto文件强约束 |

---

## 3. 内部gRPC调用

### 3.1 Proto定义（服务契约）

```protobuf
// 文件: gow.balance.base/rpc/balance.proto
syntax = "proto3";

package balance;

option go_package = "./pb";

service BalanceService {
  // 冻结用户金币（用于送礼）
  rpc FreezeUserCoinForGift(FreezeUserCoinForGiftRequest) returns (FreezeUserCoinForGiftResponse);

  // 完成交易（确认收款或退款）
  rpc CompleteTransaction(CompleteGiftTransactionRequest) returns (CompleteGiftTransactionResponse);

  // 批量增加用户币
  rpc BatchAddUserCoin(BatchAddUserCoinRequest) returns (BatchAddUserCoinResponse);
}

message FreezeUserCoinForGiftRequest {
  string from_uid = 1;   // 送礼用户ID
  string to_uid = 2;     // 收礼用户ID
  int64 coin = 3;        // 金币数量
  string biz_type = 5;   // 业务类型
  string biz_id = 6;     // 业务ID
  string scene_id = 8;   // 场景ID
}

message FreezeUserCoinForGiftResponse {
  int32 code = 1;
  string message = 2;
  string tx_id = 3;  // 交易ID
}
```

### 3.2 RPC客户端初始化（单例模式）

```go
// 文件: gow.common.lib/client/balance/rpc/client.go
package balanceservice

import (
    "sync"
    "sync/atomic"
    "github.com/zeromicro/go-zero/zrpc"
    "google.golang.org/grpc"
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Client 统一管理与余额相关服务的RPC客户端
type Client struct {
    balanceRpcConf zrpc.RpcClientConf
    balanceRpc     atomic.Value  // 存储单例客户端
    balanceRpcMu   sync.Mutex    // 保证线程安全
}

// NewClient 创建RPC客户端实例
func NewClient(balanceRpcConf zrpc.RpcClientConf) *Client {
    return &Client{
        balanceRpcConf: balanceRpcConf,
    }
}

// GetBalanceRpcClient 获取客户端（双重检查锁 + 单例模式）
func (c *Client) GetBalanceRpcClient() (svc.BalanceService, error) {
    // 快速路径：如果已初始化，直接返回（无锁）
    if client, ok := c.balanceRpc.Load().(svc.BalanceService); ok {
        return client, nil
    }

    // 加锁
    c.balanceRpcMu.Lock()
    defer c.balanceRpcMu.Unlock()

    // 双重检查
    if client, ok := c.balanceRpc.Load().(svc.BalanceService); ok {
        return client, nil
    }

    // 创建gRPC客户端（带链路追踪）
    client, err := zrpc.NewClient(
        conf,
        zrpc.WithDialOption(grpc.WithStatsHandler(otelgrpc.NewClientHandler())),
    )

    balanceService := svc.NewBalanceService(client)
    c.balanceRpc.Store(balanceService)  // 存储到全局
    return balanceService, nil
}
```

### 3.3 业务层调用gRPC

```go
// 文件: gow.home.busi/bussiness/balance_client.go
package bussiness

import (
    "context"
    "fmt"
    balancePb "gow.common.lib/client/balance/rpc/pb"
    svc "gow.common.lib/client/balance/rpc/balanceservice"
)

var globalBalanceRpcClient svc.BalanceService
var balanceRpcOnce sync.Once

// CreateBalanceRPCClient 创建余额服务RPC客户端
func CreateBalanceRPCClient() (svc.BalanceService, error) {
    var globalErr error
    balanceRpcOnce.Do(func() {
        // 从配置读取RPC服务地址
        var rpcClientConf zrpc.RpcClientConf
        rpcClientConf.Target = strings.Join(setting.CfgGlobal.BalanceRpc.Endpoints, ",")
        rpcClientConf.Timeout = int64(setting.CfgGlobal.BalanceRpc.Timeout)

        // 创建客户端
        balanceClient := balanceservice.NewClient(rpcClientConf)
        balanceRpcClient, err := balanceClient.GetBalanceRpcClient()
        if err != nil {
            globalErr = err
        }
        globalBalanceRpcCilent = balanceRpcClient
    })

    return globalBalanceRpcCilent, globalErr
}

// ChargeIMMessage 冻结用户金币（用于IM消息扣费）
func ChargeIMMessage(ctx context.Context, userID, targetUserID string, amount int) (string, error) {
    // 1. 获取RPC客户端（单例，全局复用）
    balanceClient, err := CreateBalanceRPCClient()
    if err != nil {
        return "", err
    }

    // 2. 构造请求参数
    req := &balancePb.FreezeUserCoinForGiftRequest{
        FromUid: userID,
        ToUid:   targetUserID,
        Coin:    int64(amount),
        BizId:   "100004",
        BizType: "im",
        SceneId: "200014",
    }

    // 3. 调用gRPC服务（看起来像本地函数调用）
    resp, err := balanceClient.FreezeUserCoinForGift(ctx, req)
    if err != nil {
        return "", fmt.Errorf("balance service error: %v", err)
    }

    // 4. 处理响应
    if resp.Code != 0 {
        return "", fmt.Errorf("insufficient balance: %s", resp.Message)
    }

    return resp.TxId, nil
}
```

### 3.4 调用流程图

```
home.busi (IM业务服务)          balance.base (余额服务)
     |                                      ^
     |  1. FreezeUserCoinForGift(req)       |
     |--------------------------------------|
     |        (冻结送礼方金币)                |
     |                                      |
     |  2. CompleteTransaction(req)         |
     |--------------------------------------|
     |        (确认交易，给主播分成)          |
```

---

## 4. gRPC高并发支持

### 4.1 服务端 - Go-Zero框架自动并发

```go
// 文件: gow.balance.base/rpc/balance.go
func main() {
    var c config.Config
    conf.MustLoad(*configFile, &c)

    // 创建服务上下文
    ctx := svc.NewServiceContext(c)

    // 创建gRPC服务器
    s := zrpc.MustNewServer(c.RpcServerConf, func(gRPC *grpc.Server) {
        pb.RegisterBalanceServiceServer(gRPC, balanceServer)
    })

    // 启动服务（Go-Zero自动处理并发）
    s.Start()
}
```

**关键特性**：
- ✅ 每个gRPC请求在独立的goroutine中处理
- ✅ 自动管理连接池和worker池
- ✅ 无需手动编写并发代码

### 4.2 批量操作支持

```go
// Proto定义
message BatchAddUserCoinRequest {
    repeated AddCoinOperation operations = 1;  // 可传多个操作
    string scene_id = 2;
}

// 服务端批量处理
func (l *BatchAddUserCoinLogic) BatchAddUserCoin(in *pb.BatchAddUserCoinRequest) (*pb.BatchAddUserCoinResponse, error) {
    // 遍历所有操作（并发处理）
    for _, op := range in.Operations {
        // 处理每个操作
        refundOp := commonLogic.UserCoinAtomicOperation{
            UserID:        op.ToUid,
            AmountInCents: amount,
            CoinType:      coinType,
        }
        flatOps = append(flatOps, refundOp)
    }

    // 使用Redis Lua脚本原子性执行批量操作
    resp, execErr := l.base.SvcCtx.Redis.EvalCtx(l.base.Ctx, l.luaScript, keys, args...)

    return &pb.BatchAddUserCoinResponse{Code: commonErr.CodeSuccess}, nil
}
```

**并发优势**：一次RPC调用处理**多个用户的加币操作**，减少网络往返

### 4.3 客户端连接池（单例复用）

```go
// 全局单例，所有goroutine共享
var globalBalanceRpcClient svc.BalanceService
var balanceRpcOnce sync.Once

// 双重检查锁实现
func (c *Client) GetBalanceRpcClient() (svc.BalanceService, error) {
    // 快速路径（无锁）
    if client, ok := c.balanceRpc.Load().(svc.BalanceService); ok {
        return client, nil
    }

    // 慢速路径（加锁初始化）
    c.balanceRpcMu.Lock()
    defer c.balanceRpcMu.Unlock()

    // 双重检查
    if client, ok := c.balanceRpc.Load().(svc.BalanceService); ok {
        return client, nil
    }

    // 创建客户端（只执行一次）
    client, err := zrpc.NewClient(conf, zrpc.WithDialOption(...))
    c.balanceRpc.Store(client)
    return client, nil
}
```

**并发优势**：
- ✅ 全局单例，所有goroutine共享一个连接
- ✅ 无锁快速路径（已初始化后）
- ✅ HTTP/2多路复用（一条连接处理多个并发请求）

### 4.4 实际业务调用示例

```go
// HTTP Handler中调用gRPC（高并发场景）
func CheckChatPermissionHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // 获取gRPC客户端（单例，线程安全）
    balanceRpcClient, err := bussiness.CreateBalanceRPCClient()

    // 并发调用gRPC查询余额
    coinResp, err := balanceRpcClient.GetFamilyAssetAndUserCoin(ctx, &balancePb.GetFamilyAssetAndUserCoinRequest{
        UserId:   fromUserID,
        FamilyId: "0",
    })

    chatGiftBalance = coinResp.UserCoin
}
```

**并发场景**：每个HTTP请求都会调用gRPC，但**连接是复用的**

---

## 5. 部署架构

### 5.1 Kubernetes Ingress配置

```yaml
# 文件: gow.common.lib/k8s/test/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: playpop-test-api
  namespace: test
spec:
  ingressClassName: nginx
  rules:
    - host: test-api.playpop.cc
      http:
        paths:
          # 余额服务
          - backend:
              service:
                name: gow-balance-base-api-test
                port:
                  number: 17013
            path: /woolive/balanceservice
            pathType: Prefix

          # 礼物服务
          - backend:
              service:
                name: gow-gift-base-api-test
                port:
                  number: 17008
            path: /woolive/giftservice
            pathType: Prefix

          # 用户服务
          - backend:
              service:
                name: gow-user-base-test-service
                port:
                  number: 17006
            path: /woolive/userservice
            pathType: Prefix

          # 主业务（默认路由）
          - backend:
              service:
                name: gow-home-busi-test-service
                port:
                  number: 19001
            path: /woolive
            pathType: Prefix

          # 直播服务
          - backend:
              service:
                name: gow-live-base-test-service
                port:
                  number: 17005
            path: /gow
            pathType: Prefix
```

### 5.2 服务路由规则

| 路径 | 目标服务 | 用途 |
|------|----------|------|
| `/woolive/balanceservice` | balance.base (17013) | 余额API |
| `/woolive/giftservice` | gift.base (17008) | 礼物API |
| `/woolive/userservice` | user.base (17006) | 用户API |
| `/woolive/sse` | sse.base (17012) | 推送服务 |
| `/woolive/play` | play.busi (18008) | 游戏业务 |
| `/woolive` | home.busi (19001) | 主业务（默认） |
| `/gow` | live.base (17005) | 直播服务 |

---

## 6. 代码示例

### 6.1 配置文件示例

```yaml
# gow.balance.base/rpc/etc/local/balance-rpc.yaml
Name: gow.balance.base.rpc
ListenOn: 0.0.0.0:17010
Env: local
Timeout: 300000  # 5分钟超时

Mysql:
  DataSource: root:password@tcp(host:3306)/woolive?charset=utf8mb4

Cache:
  - Host: host:6515
    Type: node
    Pass: password
```

### 6.2 服务配置结构

```go
// gow.balance.base/rpc/internal/config/config.go
package config

import (
    "github.com/zeromicro/go-zero/core/stores/cache"
    "github.com/zeromicro/go-zero/zrpc"
)

type Config struct {
    zrpc.RpcServerConf  // gRPC服务配置（内置）

    Mysql struct {
        DataSource string
    }

    Cache []cache.CacheConf  // Redis缓存配置

    UserServiceClient struct {
        BaseUrl string
    }

    Env string
}
```

### 6.3 多gRPC服务并发调用

```go
// 同时调用Balance和Gift服务
func sendChatGiftViaGiftService(ctx context.Context, fromUserID, toUserID, messageID string) error {
    // 初始化Gift服务RPC客户端
    var rpcClientConf zrpc.RpcClientConf
    rpcClientConf.Target = strings.Join(setting.CfgGlobal.GiftRpc.Endpoints, ",")
    giftClient := giftservice.NewClient(rpcClientConf)
    giftRpcClient, err := giftClient.GetGiftRpcClient()

    // 调用Gift服务
    giftReq := &pb.SendGiftRequest{
        GiftId:   chatGiftID,
        ToUserId: []string{toUserID},
        Type:     1,
    }

    giftRsp, err := giftRpcClient.SendGift(ctx, giftReq)
    return err
}
```

---

## 7. 核心技术总结

| 特性 | 实现方式 | 代码位置 |
|------|----------|----------|
| 服务端并发 | Go-Zero自动为每个请求创建goroutine | `balance.go:41` |
| 批量操作 | Proto `repeated` 字段支持 | `balance.proto:36` |
| 连接复用 | `atomic.Value` + 双重检查锁 | `client.go:32-74` |
| HTTP/2多路复用 | gRPC底层自动支持 | - |
| 链路追踪 | OpenTelemetry自动追踪 | `client.go:58` |

---

## 8. 高并发能力对比

| 维度 | HTTP/JSON | gRPC |
|------|-----------|------|
| 序列化 | 文本JSON，慢 | 二进制Protobuf，快3-5倍 |
| 连接复用 | HTTP/1.1 每请求独立连接 | HTTP/2 多路复用 |
| 并发模型 | 每个请求一个goroutine | 同左，但更轻量 |
| 批量操作 | 需要手动实现 | Proto原生支持`repeated` |
| 类型安全 | 运行时检查 | 编译时检查 |

---

## 9. 总结

### 9.1 架构设计原则

1. **对外HTTP，对内gRPC** - 兼顾兼容性和性能
2. **服务自治** - 每个服务可独立部署和扩展
3. **分层路由** - Ingress网关统一对外暴露
4. **连接复用** - 单例模式避免频繁建连

### 9.2 gRPC高并发优势

是的，**gRPC天然支持高并发**，体现在：

1. ✅ **服务端**：Go-Zero框架自动为每个gRPC请求创建独立goroutine处理
2. ✅ **客户端**：单例连接池，所有请求共享连接，无锁快速路径
3. ✅ **网络层**：HTTP/2多路复用，一条连接处理多个并发请求
4. ✅ **批量操作**：一次RPC调用处理多个业务操作，减少网络开销
5. ✅ **二进制协议**：Protobuf序列化比JSON快3-5倍，减少CPU和内存占用

项目中成千上万的HTTP请求都会触发gRPC调用，但因为有这些优化，性能依然很高！

---

**文档版本**: v1.0
**最后更新**: 2025-01-16
