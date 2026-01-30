# GoIM 即时通讯系统 - 技术文档

## 项目概述

### 基本信息
- **项目名称**: GoIM v2.0
- **项目类型**: 分布式即时通讯系统
- **开发语言**: Go (后端) + Vue 3 + TypeScript (前端)
- **架构模式**: 微服务架构
- **开源协议**: MIT License

### 技术栈

#### 后端技术栈
| 组件 | 技术 | 说明 |
|------|------|------|
| Web框架 | Gin | HTTP API服务 |
| WebSocket | Gorilla WebSocket | 实时通讯 |
| 服务发现 | Bilibili Discovery | 微服务注册与发现 |
| 消息队列 | Kafka | 异步消息推送 |
| 缓存 | Redis | 会话管理、在线状态 |
| 数据库 | MySQL | 持久化存储 |
| gRPC | Google gRPC | 服务间通信 |
| AI集成 | OpenAI API | AI聊天功能 |

#### 前端技术栈
| 组件 | 技术 | 说明 |
|------|------|------|
| 框架 | Vue 3 | 组合式API |
| 状态管理 | Pinia | 响应式状态管理 |
| UI组件 | Element Plus | 企业级UI库 |
| 构建工具 | Vite | 快速开发构建 |
| 语言 | TypeScript | 类型安全 |
| WebSocket | 原生WebSocket | 实时通讯 |

---

## 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Web前端     │  │  移动端      │  │  第三方应用  │          │
│  │  (Vue 3)     │  │  (SDK)       │  │  (API)       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                         接入层                                   │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              Comet Server (WebSocket)                │       │
│  │  - 长连接管理                                        │       │
│  │  - 消息推送                                          │       │
│  │  - 心跳检测                                          │       │
│  │  - 房间管理                                          │       │
│  └──────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                         业务层                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Logic Server │  │ ChatAPI      │  │ Job Server   │          │
│  │ - 逻辑处理   │  │ - HTTP API   │  │ - 离线消息   │          │
│  │ - 消息路由   │  │ - 用户管理   │  │ - 定时推送   │          │
│  │ - 负载均衡   │  │ - 好友系统   │  │ - 消息重试   │          │
│  │              │  │ - 群组系统   │  │              │          │
│  │              │  │ - AI集成     │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                         基础设施层                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ MySQL    │  │ Redis    │  │ Kafka    │  │Discovery │        │
│  │ 持久化   │  │ 缓存     │  │ 消息队列 │  │ 服务发现 │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
                            ↓
                    ┌──────────────┐
                    │ OpenAI API   │
                    │ AI服务       │
                    └──────────────┘
```

### 核心模块说明

#### 1. Comet Server (接入服务)
- **职责**: 维护客户端长连接，处理实时消息推送
- **端口**:
  - TCP: 3101
  - WebSocket: 3102
  - WebSocket Secure: 3103
- **功能**:
  - 客户端连接管理
  - 心跳保活
  - 房间(频道)管理
  - 消息实时推送
  - 在线状态维护

#### 2. Logic Server (逻辑服务)
- **职责**: 核心业务逻辑处理
- **端口**:
  - HTTP: 3111
  - gRPC: 3119
- **功能**:
  - 消息路由转发
  - 负载均衡策略
  - 节点管理
  - 与Comet通信协调

#### 3. ChatAPI (API服务)
- **职责**: 提供HTTP REST API
- **端口**: 3112
- **功能**:
  - 用户认证与授权 (JWT)
  - 用户管理 (注册、登录、资料)
  - 好友系统 (添加、删除、列表)
  - 群组系统 (创建、管理、成员)
  - 消息历史查询
  - **AI聊天功能**

#### 4. Job Server (任务服务)
- **职责**: 异步任务处理
- **功能**:
  - 离线消息推送
  - 定时任务调度
  - 消息重试机制
  - 广播消息处理

#### 5. Discovery (服务发现)
- **技术**: Bilibili Discovery
- **端口**: 7171
- **功能**:
  - 服务注册与发现
  - 健康检查
  - 负载均衡

---

## 数据库设计

### 核心表结构

```sql
-- 用户表
users (id, username, password_hash, nickname, avatar, signature, status, created_at, updated_at)

-- 好友关系表
friends (id, user_id, friend_id, nickname, remark, status, created_at, updated_at)

-- 好友申请表
friend_requests (id, from_user_id, to_user_id, message, status, created_at, updated_at)

-- 群组表
groups (id, group_id, name, description, avatar, owner_id, max_members, created_at, updated_at)

-- 群组成员表
group_members (id, group_id, user_id, role, nickname, muted_until, created_at, updated_at)

-- 会话表
conversations (id, user_id, conversation_id, conversation_type, unread_count, pinned, muted, created_at, updated_at)

-- 消息表
messages (id, msg_id, from_user_id, conversation_id, conversation_type, msg_type, content, seq, created_at, updated_at)

-- AI机器人表
ai_bots (id, bot_id, user_id, name, personality, model_name, temperature, max_tokens, created_at, updated_at)

-- AI对话表
ai_conversations (id, user_id, bot_id, title, created_at, updated_at)
```

### 消息类型 (conversation_type)
| 类型值 | 说明 |
|--------|------|
| 1 | 单聊 (好友聊天) |
| 2 | 群聊 |
| 3 | AI聊天 |

---

## 核心功能模块

### 1. 用户系统

#### 注册登录
- 用户名/密码注册
- JWT Token认证
- 自动续期机制
- 登录状态持久化

#### 个人资料
- 昵称、头像、个性签名
- 资料实时更新
- 头像上传 (本地存储)

### 2. 好友系统

#### 好友管理
- 发送好友申请
- 接受/拒绝申请
- 好友列表展示
- 删除好友
- 好友备注

#### 好友聊天
- 一对一实时消息
- 消息历史查询
- 未读消息计数
- 输入状态提示

### 3. 群组系统

#### 群组管理
- 创建群组
- 群组设置 (名称、头像、描述)
- 成员管理
- 角色权限 (群主、管理员、普通成员)
- 成员昵称设置
- 禁言功能
- 转让群主
- 解散群组

#### 群组聊天
- 群消息实时推送
- @成员功能
- 消息历史分页加载
- 成员在线状态

### 4. 消息系统

#### 消息类型
- 文本消息
- 图片消息 (支持上传)
- 文件消息
- 系统消息

#### 消息特性
- 实时推送 (WebSocket)
- 离线消息保存
- 消息持久化
- 消息序列号 (seq) 用于分页
- 消息已读状态
- 消息撤回

---

## AI聊天模块详解

### 模块概述

AI聊天模块是GoIM v2.0新增的功能，允许用户与具有不同"人格"的AI助手进行对话。

### 架构设计

```
┌──────────────────────────────────────────────────────────────┐
│                        前端层                                 │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐     │
│  │ AIChat.vue │  │ BotList.vue│  │  aiStore (Pinia)   │     │
│  │ 聊天界面   │  │ 机器人列表 │  │  状态管理          │     │
│  └────────────┘  └────────────┘  └────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
                          ↓ HTTP API
┌──────────────────────────────────────────────────────────────┐
│                       后端API层                               │
│  POST /ai/chat/send → handleSendAIMessage()                  │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│                       业务逻辑层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ BotManager   │  │ContextManager│  │ OpenAI       │       │
│  │ 机器人管理   │  │ 对话上下文   │  │ AI服务       │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────────┐
│                       数据持久层                              │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │ MySQL        │  │ LocalStorage │                         │
│  │ 消息存储     │  │ 前端缓存     │                         │
│  └──────────────┘  └──────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

### 预置AI机器人

系统提供4个预置的AI助手，每个都有独特的人格设定：

| BotID | 名称 | 人格类型 | 特点 | 颜色 |
|-------|------|----------|------|------|
| 9001 | 智能助手 | assistant | 有帮助、知识渊博、礼貌 | 蓝色 |
| 9002 | 聊天伙伴 | companion | 友好、共情、有趣 | 绿色 |
| 9003 | 学习导师 | tutor | 知识渊博、耐心、鼓励性 | 橙色 |
| 9004 | 创意助手 | creative | 创意、启发性、原创性 | 红色 |

### 核心组件详解

#### 1. 前端状态管理 (aiStore)

**位置**: `web/src/store/ai.ts`

```typescript
// 状态
bots: AIBot[]                    // 可用机器人列表
currentBot: AIBot | null         // 当前选中的机器人
messages: Record<number, AIMessage[]>  // 每个机器人的聊天记录
loading: boolean                 // 加载状态
sending: boolean                 // 发送状态

// 核心方法
loadBots()                       // 加载机器人列表
sendMessage(botId, message)      // 发送消息给AI
getBotMessages(botId)            // 获取机器人的消息
setCurrentBot(bot)               // 设置当前机器人
clearMessages(botId)             // 清空聊天记录
loadHistory(botId)               // 从服务器加载历史
loadMoreMessages(botId)          // 分页加载更多历史
```

#### 2. 对话上下文管理 (ContextManager)

**位置**: `internal/ai/context.go`

```go
type ContextManager struct {
    contexts map[string]*ConversationContext  // key: "botID:userID"
    ttl       time.Duration                    // 会话过期时间
    mu        sync.RWMutex
}

// 功能
- 维护每个用户与每个机器人的对话上下文
- 保存最近20条消息作为AI上下文
- 自动清理30分钟不活动的对话
- 线程安全的并发访问
```

#### 3. AI服务接口 (Service)

**位置**: `internal/ai/service.go`

```go
type Service interface {
    // 发送消息获取回复
    Chat(ctx, botID, personality, history, userMessage) -> (string, error)

    // 流式响应 (待实现)
    StreamChat(ctx, botID, personality, history, userMessage, callback) -> error

    // 动态配置
    SetModel(model)
    SetTemperature(temp)
}
```

#### 4. OpenAI实现

**位置**: `internal/ai/openai.go`

```go
type OpenAI struct {
    config     *Config
    httpClient *http.Client
}

// 调用OpenAI API
POST {BaseURL}/chat/completions
Headers:
  Authorization: Bearer {APIKey}
  Content-Type: application/json
Body:
{
  "model": "gpt-3.5-turbo",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant..."},
    {"role": "user", "content": "previous message 1"},
    {"role": "assistant", "content": "previous response 1"},
    ...
    {"role": "user", "content": "current message"}
  ],
  "stream": false
}
```

### 消息流程详解

#### 用户发送消息流程

```
1. 用户在AIChat.vue输入消息
   ↓
2. aiStore.sendMessage(botId, message)
   - 乐观更新UI (立即显示用户消息)
   - 保存到localStorage
   ↓
3. POST /ai/chat/send
   { bot_id: 9001, message: "你好" }
   ↓
4. handleSendAIMessage()
   - 验证用户身份
   - 获取机器人人格配置
   ↓
5. ContextManager.GetContext(botId, userId)
   - 获取或创建对话上下文
   - 获取最近10条历史消息
   ↓
6. BotManager.Chat()
   - 构建system prompt (根据人格)
   - 拼接历史消息
   - 调用OpenAI API
   ↓
7. OpenAI.Chat()
   - HTTP POST请求
   - 解析响应
   - 返回AI回复
   ↓
8. 保存到数据库
   - 用户消息: from_user_id=userId, conversation_type=3
   - AI回复: from_user_id=botId, conversation_type=3
   ↓
9. 返回给前端
   { reply: "你好！有什么可以帮助你的吗？", bot_id: 9001 }
   ↓
10. 更新UI
    - 显示AI回复
    - 更新localStorage
    - 更新会话列表预览
```

### AI人格系统

#### System Prompt构建

```go
func BuildSystemPrompt(personality *Personality) string {
    if personality.SystemPrompt != "" {
        return personality.SystemPrompt
    }

    prompt := "You are a " + personality.Role + " with a " + personality.Tone + " tone.\n"
    if len(personality.Traits) > 0 {
        prompt += "Your personality traits are: "
        for i, trait := range personality.Traits {
            if i > 0 {
                prompt += ", "
            }
            prompt += trait
        }
        prompt += ".\n"
    }
    return prompt
}
```

#### 预置人格配置

```go
var DefaultPersonalities = map[string]*Personality{
    "assistant": {
        Name:    "智能助手",
        Tone:    "friendly",
        Role:    "assistant",
        Traits:  []string{"helpful", "knowledgeable", "polite"},
        SystemPrompt: "You are a helpful AI assistant. Answer questions clearly and concisely.",
    },
    "companion": {
        Name:    "聊天伙伴",
        Tone:    "casual",
        Role:    "companion",
        Traits:  []string{"friendly", "empathetic", "fun"},
        SystemPrompt: "You are a friendly chat companion. Be engaging and supportive in casual conversation.",
    },
    "tutor": {
        Name:    "学习导师",
        Tone:    "professional",
        Role:    "tutor",
        Traits:  []string{"knowledgeable", "patient", "encouraging"},
        SystemPrompt: "You are a patient tutor. Explain concepts clearly and encourage learning.",
    },
    "creative": {
        Name:    "创意助手",
        Tone:    "imaginative",
        Role:    "creative",
        Traits:  []string{"creative", "inspiring", "original"},
        SystemPrompt: "You are a creative assistant. Help with brainstorming and creative thinking.",
    },
}
```

### 数据存储

#### 前端缓存 (localStorage)
```javascript
key: 'ai_messages'
value: {
  "9001": [
    { role: "user", content: "你好", timestamp: 1704067200000 },
    { role: "assistant", content: "你好！有什么可以帮助你的吗？", timestamp: 1704067201000 }
  ],
  "9002": [...]
}
```

#### 后端数据库 (messages表)
```sql
-- 用户消息
INSERT INTO messages (from_user_id, conversation_id, conversation_type, content, seq)
VALUES (123, 9001, 3, '你好', 1)

-- AI回复
INSERT INTO messages (from_user_id, conversation_id, conversation_type, content, seq)
VALUES (9001, 9001, 3, '你好！有什么可以帮助你的吗？', 2)

-- conversation_type=3 表示AI聊天
```

### API接口

#### 获取机器人列表
```http
GET /ai/bots
Authorization: Bearer {jwt_token}

Response:
{
  "code": 0,
  "data": {
    "bots": [
      {
        "id": 9001,
        "name": "智能助手",
        "personality": "assistant",
        "role": "assistant",
        "tone": "friendly",
        "is_default": true
      },
      ...
    ]
  }
}
```

#### 发送消息
```http
POST /ai/chat/send
Authorization: Bearer {jwt_token}
Content-Type: application/json

Request:
{
  "bot_id": 9001,
  "message": "你好，请介绍一下你自己"
}

Response:
{
  "code": 0,
  "data": {
    "reply": "你好！我是智能助手，我可以帮助你解答问题、提供信息和进行对话。",
    "bot_id": 9001,
    "user_msg_id": 10001,
    "ai_msg_id": 10002
  }
}
```

### 配置说明

```toml
[ai]
provider      = "openai"          # AI提供商
api_key       = "sk-xxx"          # OpenAI API密钥
base_url      = "https://api.openai.com/v1"  # API地址(支持自定义)
model         = "gpt-3.5-turbo"   # 模型名称
temperature   = 0.7               # 温度参数(0-1, 越高越随机)
max_tokens    = 1000              # 最大token数
```

### 性能优化

1. **上下文管理**
   - 仅保留最近20条消息作为上下文
   - 自动清理过期会话 (30分钟)
   - 内存缓存减少数据库查询

2. **前端缓存**
   - localStorage本地消息缓存
   - 乐观更新UI提升响应速度
   - 分页加载历史消息

3. **异步处理**
   - AI请求异步处理
   - 加载状态指示器
   - 错误重试机制

### 扩展性设计

1. **多AI提供商支持**
   ```go
   type Service interface {
       Chat(...) (string, error)
   }
   // 可轻松扩展支持其他AI服务
   ```

2. **自定义机器人**
   - 用户可创建具有特定人格的AI机器人
   - 支持自定义system prompt
   - 独立的模型参数配置

3. **流式响应预留**
   ```go
   StreamChat(..., callback func(chunk string)) error
   // 预留流式响应接口
   ```

---

## WebSocket通讯协议

### 连接建立

```javascript
const ws = new WebSocket('ws://localhost:3102/sub')

ws.onopen = () => {
  // 发送认证消息
  ws.send(JSON.stringify({
    ver: 1,
    op:  7,  // 认证操作
    seq: 1,
    body: JSON.stringify({
      token: jwt_token,
      platform: "web",
      device_id: "xxx"
    })
  }))
}
```

### 操作码 (op)

| 操作码 | 名称 | 说明 |
|--------|------|------|
| 0 | 心跳 | 客户端定时发送保持连接 |
| 1 | 心跳响应 | 服务器响应心跳 |
| 2 | 消息推送 | 服务器推送新消息 |
| 3 | 消息确认 | 客户端确认收到消息 |
| 7 | 认证 | 连接认证 |
| 8 | 认证响应 | 认证结果 |

### 消息格式

```json
{
  "ver": 1,           // 协议版本
  "op": 2,            // 操作码
  "seq": 123,         // 序列号
  "body": {           // 消息体
    "msg_id": 10001,
    "from_user_id": 123,
    "conversation_id": 456,
    "conversation_type": 1,
    "content": "{\"text\":\"Hello\"}",
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

---

## 部署架构

### 开发环境

```bash
# 项目结构
goim/
├── discovery/          # 服务发现 (独立仓库)
│   └── discovery
├── goim/              # 主项目
    ├── cmd/
    │   ├── comet/     # 接入服务
    │   ├── logic/     # 逻辑服务
    │   ├── job/       # 任务服务
    │   └── chatapi/   # API服务
    ├── internal/      # 内部包
    ├── web/           # 前端
    └── Makefile       # 构建脚本

# 启动命令
make discovery-start   # 启动服务发现
make start            # 启动所有服务
make web-dev          # 启动前端
```

### 生产环境建议

```
┌─────────────────────────────────────────────────────────┐
│                    负载均衡器                            │
│                   (Nginx/SLB)                          │
└─────────────────────────────────────────────────────────┘
                            ↓
        ┌───────────────────┴───────────────────┐
        ↓                                       ↓
┌───────────────┐                     ┌───────────────┐
│ Comet集群     │                     │ ChatAPI集群   │
│ (多实例)      │                     │ (多实例)      │
└───────────────┘                     └───────────────┘
        ↓                                       ↓
┌───────────────┐                     ┌───────────────┐
│ Logic集群     ←────────────────────→│ Redis集群     │
└───────────────┘     gRPC             └───────────────┘
        ↓                                       ↓
┌───────────────┐                     ┌───────────────┐
│ Kafka集群     │                     │ MySQL主从     │
└───────────────┘                     └───────────────┘
```

---

## 配置文件

### Discovery配置 (discovery.toml)
```toml
nodes = ["127.0.0.1:7171"]  # 集群节点

[env]
region = "sh"              # 区域
zone = "sh001"             # 可用区
host = "test1"             # 主机标识
DeployEnv = "dev"          # 部署环境

[httpServer]
addr = "127.0.0.1:7171"    # 监听地址
```

### Comet配置 (comet.toml)
```toml
[discovery]
nodes = ["127.0.0.1:7171"]  # 服务发现地址

[server]
listen = "tcp://0.0.0.0:3101"     # TCP端口
websocket = "tcp://0.0.0.0:3102"  # WebSocket端口
```

### ChatAPI配置 (chatapi.toml)
```toml
[mysql]
dsn = "root:password@tcp(127.0.0.1:3306)/goim_chat"

[redis]
addr = "127.0.0.1:6379"

[jwt]
secret = "your-secret-key"
expire_time = 604800  # 7天

[ai]
provider = "openai"
api_key = "sk-xxx"
model = "gpt-3.5-turbo"
temperature = 0.7
max_tokens = 1000
```

---

## 开发指南

### 环境要求
- Go 1.21+
- Node.js 18+
- MySQL 8.0+
- Redis 6.0+
- Kafka 2.8+

### 本地开发

```bash
# 1. 启动依赖服务
make docker-up

# 2. 初始化数据库
make db-init

# 3. 编译服务
make build

# 4. 启动服务
make start

# 5. 启动前端
cd web && npm run dev
```

### 常用命令

```bash
make build         # 编译所有服务
make start         # 启动所有服务
make stop          # 停止所有服务
make restart       # 重启服务
make status        # 查看服务状态
make logs          # 查看日志
make clean         # 清理编译文件

# AI相关
make discovery-setup    # 安装discovery
make discovery-start    # 启动discovery
make discovery-stop     # 停止discovery
```

---

## 已实现功能清单

### 用户系统 ✅
- [x] 用户注册
- [x] 用户登录 (JWT)
- [x] 个人资料编辑
- [x] 头像上传
- [x] 个性签名设置
- [x] 在线状态

### 好友系统 ✅
- [x] 发送好友申请
- [x] 接受/拒绝申请
- [x] 好友列表
- [x] 删除好友
- [x] 好友备注
- [x] 好友聊天

### 群组系统 ✅
- [x] 创建群组
- [x] 群组设置
- [x] 邀请成员
- [x] 移除成员
- [x] 角色管理 (群主/管理员/成员)
- [x] 成员昵称
- [x] 禁言功能
- [x] 转让群主
- [x] 解散群组
- [x] 群组聊天

### 消息系统 ✅
- [x] 实时消息推送
- [x] 离线消息
- [x] 消息历史
- [x] 分页加载
- [x] 未读计数
- [x] 消息已读
- [x] 输入状态

### AI聊天 ✅
- [x] 多人格AI助手
- [x] 对话历史管理
- [x] 上下文记忆
- [x] 消息持久化
- [x] 流式响应接口 (预留)

### UI功能 ✅
- [x] 响应式设计
- [x] 暗色主题
- [x] 消息搜索
- [x] 会话置顶
- [x] 消息免打扰
- [x] 用户头像点击查看资料

---

## 待优化功能

### 短期优化
- [ ] 消息撤回功能
- [ ] 消息引用回复
- [ ] 图片/文件消息完善
- [ ] 群组@功能完善
- [ ] 消息搜索优化

### 长期规划
- [ ] AI流式响应实现
- [ ] 语音消息
- [ ] 视频通话
- [ ] 端到端加密
- [ ] 消息同步到云端
- [ ] 多设备同步
- [ ] 消息转发
- [ ] 群组公告
- [ ] 群组文件

---

## 常见问题

### Q1: 如何自定义AI人格？
在数据库`ai_bots`表中创建新记录，设置自定义的`personality`字段（JSON格式）。

### Q2: 如何使用兼容OpenAI的API？
修改配置文件中的`base_url`，例如：
```toml
[ai]
base_url = "https://api.deepseek.com/v1"
```

### Q3: 如何增加新的Comet实例？
直接部署新实例，配置相同的discovery地址，会自动注册到服务发现。

### Q4: 消息如何保证不丢失？
- WebSocket实时推送
- Kafka异步消息队列
- 数据库持久化
- Job离线消息重试

---

## 总结

GoIM v2.0是一个功能完整的即时通讯系统，具备以下特点：

1. **微服务架构** - 各服务独立部署，易于扩展
2. **高性能** - 使用Go语言，支持百万级并发连接
3. **实时通讯** - WebSocket长连接，消息即时推送
4. **AI集成** - 创新的AI聊天功能，支持多人格助手
5. **完整功能** - 用户、好友、群组、消息全覆盖
6. **前后端分离** - Vue 3 + TypeScript，现代化前端架构

项目代码结构清晰，注释完整，适合作为即时通讯系统的学习参考和二次开发基础。

---

## 文档信息

- **文档版本**: v1.0
- **编写日期**: 2024年1月
- **适用版本**: GoIM v2.0.2
- **维护者**: Development Team
