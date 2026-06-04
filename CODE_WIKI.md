# New API - Code Wiki

> **版本**: 基于仓库当前代码 | **语言**: Go 1.25 + TypeScript/React | **协议**: MIT

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [目录结构](#3-目录结构)
4. [核心模块详解](#4-核心模块详解)
   - [4.1 入口 (main.go)](#41-入口-maingo)
   - [4.2 路由层 (router/)](#42-路由层-router)
   - [4.3 中间件层 (middleware/)](#43-中间件层-middleware)
   - [4.4 控制器层 (controller/)](#44-控制器层-controller)
   - [4.5 数据模型层 (model/)](#45-数据模型层-model)
   - [4.6 中继适配层 (relay/)](#46-中继适配层-relay)
   - [4.7 服务层 (service/)](#47-服务层-service)
   - [4.8 公共模块 (common/)](#48-公共模块-common)
   - [4.9 常量模块 (constant/)](#49-常量模块-constant)
   - [4.10 DTO 层 (dto/)](#410-dto-层-dto)
   - [4.11 OAuth 模块 (oauth/)](#411-oauth-模块-oauth)
   - [4.12 设置模块 (setting/)](#412-设置模块-setting)
   - [4.13 工具包 (pkg/)](#413-工具包-pkg)
   - [4.14 日志模块 (logger/)](#414-日志模块-logger)
   - [4.15 国际化模块 (i18n/)](#415-国际化模块-i18n)
   - [4.16 前端 (web/)](#416-前端-web)
5. [数据模型 ER 关系](#5-数据模型-er-关系)
6. [依赖关系](#6-依赖关系)
7. [项目运行方式](#7-项目运行方式)
8. [API 路由清单](#8-api-路由清单)

---

## 1. 项目概述

**New API** 是一个下一代 LLM 网关和 AI 资产管理平台。它充当 AI API 的统一代理层，提供：

- **多供应商统一代理**: 支持 50+ 种 AI 供应商（OpenAI、Claude、Gemini、Azure、百度文心、阿里通义、讯飞星火等），对外暴露统一的 OpenAI 兼容 API
- **多租户管理**: 用户体系、令牌管理、分组/渠道管理
- **计费与配额**: 灵活的定价模型、计费表达式引擎、订阅制支持
- **负载均衡与故障切换**: 多渠道随机选择、亲和性缓存、自动重试
- **中继透明代理**: 请求/响应格式转换、模型映射、流式传输
- **异步任务系统**: 支持 Midjourney、Suno、视频生成等异步任务轮询

**技术栈**:
| 层 | 技术 |
|---|---|
| 后端 | Go + Gin + GORM |
| 默认前端 | React 19 + TypeScript + TanStack Router + shadcn/ui + Rsbuild |
| 经典前端 | React 18 + JavaScript + Semi UI + Vite |
| 桌面端 | Electron |
| 数据库 | PostgreSQL / MySQL / SQLite |
| 缓存 | Redis + 内存缓存 |

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      客户端 (Client)                         │
│         Web 浏览器 / API 客户端 / Midjourney 客户端            │
└───────────────────────┬─────────────────────────────────────┘
                        │
         ┌──────────────▼──────────────┐
         │      Gin HTTP Server       │
         │         (main.go)          │
         └──────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │      中间件 (middleware)      │
         │   CORS / 认证 / 限流 / 分发    │
         └──────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │         路由层 (router)       │
         │  /api /v1 /mj /suno /web    │
         └──────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │      控制器 (controller)      │
         │    Relay / CRUD / 计费       │
         └──────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │  中继引擎 (relay) + 适配器     │
         │  50+ AI 供应商协议适配         │
         └──────────────┬─────────────┘
                        │
         ┌──────────────▼──────────────┐
         │      上游 AI 服务商          │
         │  OpenAI / Claude / Gemini... │
         └─────────────────────────────┘
```

**请求处理流水线**:

```
客户端请求
  → CORS 中间件
  → 认证中间件 (TokenAuth / UserAuth)
  → 限流中间件 (RateLimit)
  → 分发中间件 (Distribute: 选择渠道/模型映射)
  → 控制器 Relay()
  → 中继引擎 (根据 RelayFormat 分发到对应 handler)
  → 适配器 (Adaptor: 协议转换 + 请求转发)
  → 上游 AI 服务
  → 响应回传 (计费结算 + 日志记录)
```

---

## 3. 目录结构

```
new-api/
├── main.go                    # 服务入口，初始化资源、启动Gin服务器
├── go.mod / go.sum            # Go 模块依赖
├── makefile                   # 构建/开发命令
├── Dockerfile                 # 多阶段 Docker 构建
├── docker-compose.yml         # 生产部署编排
├── docker-compose.dev.yml     # 开发环境编排
├── .env.example               # 环境变量示例
│
├── common/                    # 公共库：环境变量、加密、Redis、限流器等
├── constant/                  # 常量定义：渠道类型、API类型、上下文Key等
├── controller/                # HTTP 请求处理器（控制器）
├── docs/                      # 文档和图片资源
├── dto/                       # 数据传输对象（请求/响应结构体）
├── electron/                  # Electron 桌面端源码
├── i18n/                      # 国际化支持
├── logger/                    # 日志模块
├── middleware/                 # Gin 中间件
├── model/                     # 数据模型/GORM 实体 + 数据库初始化
├── oauth/                     # OAuth 提供者注册与实现
├── pkg/                       # 独立的工具包
│   ├── billingexpr/           # 计费表达式引擎
│   ├── cachex/                # 缓存抽象层
│   ├── ionet/                 # IO.NET 模型部署客户端
│   └── perf_metrics/          # 性能指标收集
├── relay/                     # 中继引擎（核心）
│   ├── channel/               # 各供应商适配器实现
│   ├── common/                # 中继公共信息、工具
│   ├── common_handler/        # 公共处理器
│   ├── constant/              # 中继模式常量
│   ├── helper/                # 中继辅助函数
│   ├── reasonmap/             # 推理内容映射
│   └── relay_adaptor.go       # 适配器工厂
├── router/                    # 路由注册
├── service/                   # 业务逻辑服务层
├── setting/                   # 运行时配置管理
├── types/                     # 自定义类型定义
└── web/                       # 前端源码
    ├── classic/               # 经典前端 (React 18 + Semi UI)
    └── default/               # 默认前端 (React 19 + TanStack + shadcn/ui)
```

---

## 4. 核心模块详解

### 4.1 入口 (main.go)

**文件**: [main.go](file:///workspace/main.go)

**关键函数**:

| 函数 | 职责 |
|---|---|
| `main()` | 程序入口，依次初始化所有资源，启动 Gin HTTP 服务器 |
| `InitResources()` | 初始化顺序加载：.env → 环境变量 → 日志 → 数据库 → Redis → 缓存 → 性能监控 → i18n |
| `InjectUmamiAnalytics()` | 嵌入 Umami 网站分析脚本到前端 HTML |
| `InjectGoogleAnalytics()` | 嵌入 Google Analytics 脚本 |

**启动流程**:
1. 加载 `.env` 文件
2. 初始化环境变量 (`common.InitEnv()`)
3. 初始化日志器 (`logger.SetupLogger()`)
4. 初始化模型设置 (`ratio_setting.InitRatioSettings()`)
5. 初始化 HTTP 客户端 (`service.InitHttpClient()`)
6. 初始化数据库 (`model.InitDB()`)，执行 GORM AutoMigrate
7. 初始化选项 (`model.InitOptionMap()`)
8. 初始化日志数据库 (`model.InitLogDB()`)
9. 初始化 Redis (`common.InitRedisClient()`)
10. 初始化 i18n (`i18n.Init()`)
11. 启动后台任务（渠道缓存同步、配额统计、渠道自动测试/更新等）
12. 注册路由 → 启动 HTTP 服务器

**嵌入前端资源**:
```go
//go:embed web/default/dist
var buildFS embed.FS

//go:embed web/classic/dist
var classicBuildFS embed.FS
```
编译时将前端构建产物嵌入二进制文件，实现单文件部署。

---

### 4.2 路由层 (router/)

路由层负责将 URL 路径映射到控制器和中间件。

| 文件 | 职责 |
|---|---|
| [main.go](file:///workspace/router/main.go) | `SetRouter()` 组合所有子路由 |
| [api-router.go](file:///workspace/router/api-router.go) | `/api/*` 管理 API 路由（用户、渠道、令牌、日志等） |
| [relay-router.go](file:///workspace/router/relay-router.go) | `/v1/*`, `/mj/*`, `/suno/*` 中继路由 |
| [dashboard.go](file:///workspace/router/dashboard.go) | 仪表盘 SSR 路由 |
| [video-router.go](file:///workspace/router/video-router.go) | 视频任务路由 |
| [web-router.go](file:///workspace/router/web-router.go) | 前端 SPA 静态文件服务 |

**路由分组结构**:

```
/api              → 管理 API（认证、用户、渠道、令牌、日志等）
  /api/setup      → 初始化向导
  /api/user/*     → 用户 CRUD + OAuth 登录
  /api/channel/*  → 渠道管理
  /api/token/*    → 令牌管理
  /api/log/*      → 使用日志查询
  /api/usage/*    → 用量查询
  /api/option/*   → 系统选项（Root权限）
  /api/redemption/* → 兑换码管理
  /api/subscription/* → 订阅管理

/v1               → OpenAI 兼容 API 中继
  /v1/chat/completions    → 聊天补全
  /v1/completions         → 文本补全
  /v1/responses           → Responses API
  /v1/images/generations  → 图像生成
  /v1/embeddings          → 文本嵌入
  /v1/audio/*             → 语音转写/合成
  /v1/models/*            → 模型列表
  /v1/messages            → Claude Messages API
  /v1/rerank              → 重排序

/v1beta/models/*   → Gemini 原生 API
/mj/*              → Midjourney API
/suno/*            → Suno 音乐生成 API
/pg/*              → Playground (仪表盘内嵌)
```

---

### 4.3 中间件层 (middleware/)

中间件按请求处理顺序执行，提供横切关注点的实现。

| 文件 | 职责 | 执行顺序 |
|---|---|---|
| [cors.go](file:///workspace/middleware/cors.go) | 跨域 CORS 处理 | 1 |
| [request-id.go](file:///workspace/middleware/request-id.go) | 注入 `X-Oneapi-Request-Id` | 2 |
| [i18n.go](file:///workspace/middleware/i18n.go) | 设置请求语言上下文 | 3 |
| [logger.go](file:///workspace/middleware/logger.go) | 请求日志记录 | 4（SetupLogger） |
| [auth.go](file:///workspace/middleware/auth.go) | 认证/鉴权 | 5 |
| [rate-limit.go](file:///workspace/middleware/rate-limit.go) | 全局限流 | 6 |
| [model-rate-limit.go](file:///workspace/middleware/model-rate-limit.go) | 模型级别限流 | 7 |
| [distributor.go](file:///workspace/middleware/distributor.go) | 渠道分发/模型选择 | 8（核心） |
| [stats.go](file:///workspace/middleware/stats.go) | 请求统计 | 9 |
| [gzip.go](file:///workspace/middleware/gzip.go) | GZIP 压缩 | 10 |
| [recover.go](file:///workspace/middleware/recover.go) | Panic 恢复 | 全局 |
| [secure_verification.go](file:///workspace/middleware/secure_verification.go) | 安全验证（敏感操作） | 按需 |
| [turnstile-check.go](file:///workspace/middleware/turnstile-check.go) | Cloudflare Turnstile 人机验证 | 按需 |
| [disable-cache.go](file:///workspace/middleware/disable-cache.go) | 禁用缓存头 | 按需 |
| [cache.go](file:///workspace/middleware/cache.go) | 响应缓存 | 按需 |
| [jimeng_adapter.go](file:///workspace/middleware/jimeng_adapter.go) | 即梦适配中间件 | 按需 |
| [kling_adapter.go](file:///workspace/middleware/kling_adapter.go) | 可灵适配中间件 | 按需 |
| [performance.go](file:///workspace/middleware/performance.go) | 系统性能检查 | 中继路由 |

**认证中间件层次**:

| 中间件 | 最低角色 | 使用场景 |
|---|---|---|
| `TryUserAuth()` | 无需登录 | 尝试读取会话 |
| `UserAuth()` | CommonUser (1) | 普通用户接口 |
| `AdminAuth()` | AdminUser (10) | 管理接口 |
| `RootAuth()` | RootUser (100) | 根用户接口 |
| `TokenAuth()` | - | API Token 认证（中继路由） |
| `TokenAuthReadOnly()` | - | 只读 Token 认证（用量查询） |
| `TokenOrUserAuth()` | - | 兼容两种认证方式 |

**Distribute 中间件（核心）**:

[distributor.go](file:///workspace/middleware/distributor.go) 是中继请求的核心分发逻辑：

1. 从请求中提取 `model` 参数（支持 JSON/multipart/form-urlencoded）
2. 检查 Token 模型限制 (`ContextKeyTokenModelLimit`)
3. 尝试渠道亲和性优先匹配 (`GetPreferredChannelByAffinity`)
4. 如果无亲和渠道，随机选取一个可用渠道 (`CacheGetRandomSatisfiedChannel`)
5. 设置渠道上下文信息（ID、类型、密钥、BaseURL、参数覆盖等）
6. 请求成功后记录渠道亲和性 (`RecordChannelAffinity`)

**特殊路由处理**:
- `/v1beta/models/*` → Gemini 原生 API 路径
- `/v1/messages` → Claude Messages API (支持 `x-api-key` 头)
- `/mj/*` → Midjourney API
- `/suno/*` → Suno API
- `/v1/realtime` → OpenAI Realtime WebSocket

---

### 4.4 控制器层 (controller/)

控制器负责处理 HTTP 请求，解析参数，调用服务层，返回响应。

**核心文件**:

| 文件 | 职责 |
|---|---|
| [relay.go](file:///workspace/controller/relay.go) | **中继主入口**：根据 `RelayFormat` 分发到对应中继处理器 |
| [channel.go](file:///workspace/controller/channel.go) | 渠道 CRUD、批量操作、模型同步 |
| [token.go](file:///workspace/controller/token.go) | 令牌 CRUD |
| [user.go](file:///workspace/controller/user.go) | 用户注册/登录/管理 |
| [billing.go](file:///workspace/controller/billing.go) | 计费相关 |
| [channel-billing.go](file:///workspace/controller/channel-billing.go) | 渠道计费结算 |
| [log.go](file:///workspace/controller/log.go) | 使用日志查询 |
| [option.go](file:///workspace/controller/option.go) | 系统选项管理 |
| [midjourney.go](file:///workspace/controller/midjourney.go) | Midjourney 任务管理 |
| [task.go](file:///workspace/controller/task.go) | 异步任务管理 |
| [subscription.go](file:///workspace/controller/subscription.go) | 订阅管理 |
| [topup.go](file:///workspace/controller/topup.go) | 充值管理 |
| [redemption.go](file:///workspace/controller/redemption.go) | 兑换码管理 |
| [oauth.go](file:///workspace/controller/oauth.go) | OAuth 登录路由 |
| [oidc.go](file:///workspace/controller/oidc.go) | OIDC 登录 |
| [github.go](file:///workspace/controller/github.go) | GitHub OAuth |
| [discord.go](file:///workspace/controller/discord.go) | Discord OAuth |
| [linuxdo.go](file:///workspace/controller/linuxdo.go) | LinuxDO OAuth |
| [wechat.go](file:///workspace/controller/wechat.go) | 微信登录 |
| [telegram.go](file:///workspace/controller/telegram.go) | Telegram 登录 |
| [passkey.go](file:///workspace/controller/passkey.go) | WebAuthn/Passkey 认证 |
| [twofa.go](file:///workspace/controller/twofa.go) | 两步认证 |
| [model.go](file:///workspace/controller/model.go) | 模型元数据管理 |
| [group.go](file:///workspace/controller/group.go) | 分组管理 |
| [pricing.go](file:///workspace/controller/pricing.go) | 定价展示 |
| [checkin.go](file:///workspace/controller/checkin.go) | 签到 |
| [playground.go](file:///workspace/controller/playground.go) | Playground 聊天功能 |
| [setup.go](file:///workspace/controller/setup.go) | 系统初始化向导 |
| [usedata.go](file:///workspace/controller/usedata.go) | 用量数据统计 |
| [image.go](file:///workspace/controller/image.go) | 图片处理 |
| [channel-test.go](file:///workspace/controller/channel-test.go) | 渠道连通性测试 |
| [video_proxy.go](file:///workspace/controller/video_proxy.go) | 视频代理 |
| [performance.go](file:///workspace/controller/performance.go) | 性能监控接口 |
| [deployment.go](file:///workspace/controller/deployment.go) | IO.NET 模型部署管理 |
| [misc.go](file:///workspace/controller/misc.go) | 杂项 API（状态、通知等） |

**Relay 核心流程** ([relay.go](file:///workspace/controller/relay.go)):
1. 接收中继请求（通过 `RelayFormat` 区分请求类型）
2. 构建 `RelayInfo` 上下文（包含请求、渠道、用户信息）
3. 根据 `RelayFormat` 分发：
   - `RelayFormatOpenAI` → 聊天/补全/审核
   - `RelayFormatClaude` → Claude Messages
   - `RelayFormatGemini` → Gemini Native
   - `RelayFormatOpenAIImage` → 图像生成
   - `RelayFormatEmbedding` → 嵌入
   - `RelayFormatOpenAIAudio` → 音频
   - `RelayFormatRerank` → 重排序
   - `RelayFormatOpenAIResponses` → Responses API
4. 调用适配器进行协议转换和转发
5. 记录使用日志和计费信息

---

### 4.5 数据模型层 (model/)

数据模型层使用 GORM 进行 ORM 映射，支持 PostgreSQL/MySQL/SQLite。

**核心模型**:

| 模型 | 表名 | 字段概要 |
|---|---|---|
| `User` | users | 用户名、密码、角色、配额、分组、状态 |
| `Channel` | channels | 类型、密钥、BaseURL、模型列表、状态、权重、参数覆盖 |
| `Token` | tokens | Key、用户ID、名称、剩余配额、分组、模型限制、过期时间 |
| `Ability` | abilities | 渠道ID、模型名、渠道类型、是否启用 |
| `Log` | logs | 用户ID、令牌名、模型名、渠道ID、消耗Token数、配额消耗 |
| `Option` | options | Key-Value 系统配置 |
| `Redemption` | redemptions | 兑换码、配额、状态、使用用户 |
| `Midjourney` | midjourneys | MJ任务ID、状态、图片URL、操作类型 |
| `Task` | tasks | 异步任务（视频/音乐生成）、平台、状态、结果 |
| `TopUp` | topups | 充值订单、金额、支付方式、状态 |
| `QuotaData` | quota_datas | 按日期统计的配额数据 |
| `Model` | models | 模型元数据（名称、所有者、描述） |
| `Vendor` | vendors | 供应商元数据 |
| `Setup` | setups | 系统初始化状态 |
| `TwoFA` | twofas | 两步认证信息 |
| `TwoFABackupCode` | twofa_backup_codes | 2FA备用码 |
| `Checkin` | checkins | 签到记录 |
| `PasskeyCredential` | passkey_credentials | WebAuthn凭证 |
| `SubscriptionPlan` | subscription_plans | 订阅套餐 |
| `SubscriptionOrder` | subscription_orders | 订阅订单 |
| `UserSubscription` | user_subscriptions | 用户订阅关系 |
| `SubscriptionPreConsumeRecord` | subscription_pre_consume_records | 订阅预消费记录 |
| `CustomOAuthProvider` | custom_oauth_providers | 自定义OAuth提供商 |
| `UserOAuthBinding` | user_oauth_bindings | 用户OAuth绑定 |
| `PrefillGroup` | prefill_groups | 预填分组 |
| `PerfMetric` | perf_metrics | 性能指标 |

**数据库初始化** ([model/main.go](file:///workspace/model/main.go)):

| 函数 | 职责 |
|---|---|
| `InitDB()` | 初始化主数据库，执行 AutoMigrate |
| `InitLogDB()` | 初始化日志数据库（可与主库分离） |
| `CheckSetup()` | 检查系统初始化状态 |
| `createRootAccountIfNeed()` | 首次启动自动创建 root 用户 |
| `chooseDB()` | 根据 DSN 选择数据库驱动 |
| `CloseDB()` | 关闭数据库连接 |
| `PingDB()` | 数据库健康检查（10秒缓存） |

**数据库驱动选择**:
- DSN 以 `postgres://` 开头 → PostgreSQL
- DSN 以 `local` 开头 → SQLite
- 其他 → MySQL
- DSN 为空 → SQLite（默认）

**Channel 模型关键字段** ([model/channel.go](file:///workspace/model/channel.go)):
- `Type`: 渠道类型（对应供应商，如 OpenAI=1, Claude=14）
- `Key`: API 密钥（支持多 Key 用换行分隔）
- `BaseURL`: 自定义 API 地址
- `Models`: 支持的模型列表
- `ModelMapping`: 模型名映射规则
- `ParamOverride`: 参数覆盖配置
- `StatusCodeMapping`: 状态码映射（用于自动禁用）
- `Priority`: 优先级（用于渠道选择排序）
- `Weight`: 权重（用于加权随机选择）
- `AutoBan`: 自动禁用阈值

---

### 4.6 中继适配层 (relay/)

中继层是项目的核心，负责 OpenAI 格式请求到各供应商原生协议的转换。

**适配器工厂** ([relay/relay_adaptor.go](file:///workspace/relay/relay_adaptor.go)):

`GetAdaptor(apiType)` 根据 API 类型返回对应的适配器实例。

**支持的供应商适配器** (relay/channel/):

| 适配器目录 | 供应商 | 适配器目录 | 供应商 |
|---|---|---|---|
| openai/ | OpenAI | ali/ | 阿里通义千问 |
| claude/ | Anthropic Claude | baidu/ | 百度文心 |
| gemini/ | Google Gemini | zhipu/ | 智谱 GLM |
| azure/ | Azure OpenAI | zhipu_4v/ | 智谱 GLM-4V |
| aws/ | AWS Bedrock | xunfei/ | 讯飞星火 |
| cohere/ | Cohere | tencent/ | 腾讯混元 |
| vertex/ | Google Vertex AI | minimax/ | MiniMax |
| ollama/ | Ollama | moonshot/ | 月之暗面 |
| deepseek/ | DeepSeek | mistral/ | Mistral |
| perplexity/ | Perplexity | volcengine/ | 火山引擎 |
| cloudflare/ | Cloudflare | siliconflow/ | 硅基流动 |
| jina/ | Jina AI | lingyiwanwu/ | 零一万物 |
| dify/ | Dify | coze/ | 扣子 |
| xai/ | xAI (Grok) | jimeng/ | 即梦 |
| replicate/ | Replicate | codex/ | Codex (OpenAI) |
| submodel/ | Submodel | mokaai/ | Moka AI |
| palm/ | Google PaLM | baidu_v2/ | 百度千帆 V2 |
| ai360/ | 360 AI | xinference/ | Xinference |

**任务适配器** (relay/channel/task/):

| 适配器 | 平台 | 用途 |
|---|---|---|
| suno/ | Suno | 音乐生成 |
| kling/ | 可灵 | 视频生成 |
| jimeng/ | 即梦 | 视频生成 |
| ali/ | 阿里 | 视频生成 |
| doubao/ | 豆包 | 视频生成 |
| gemini/ | Gemini | 视频生成 |
| sora/ | Sora | 视频生成 |
| hailuo/ | 海螺 | 视频生成 |
| vertex/ | Vertex AI | 视频生成 |
| vidu/ | Vidu | 视频生成 |

**Adaptor 接口** ([relay/channel/adapter.go](file:///workspace/relay/channel/adapter.go)):

```go
type Adaptor interface {
    Init(info *RelayInfo)
    GetRequestURL(info *RelayInfo) (string, error)
    SetupRequestHeader(c *gin.Context, req *http.Header, info *RelayInfo) error
    ConvertOpenAIRequest(c *gin.Context, info *RelayInfo, request *GeneralOpenAIRequest) (any, error)
    ConvertRerankRequest(c *gin.Context, relayMode int, request RerankRequest) (any, error)
    ConvertEmbeddingRequest(c *gin.Context, info *RelayInfo, request EmbeddingRequest) (any, error)
    ConvertAudioRequest(c *gin.Context, info *RelayInfo, request AudioRequest) (io.Reader, error)
    ConvertImageRequest(c *gin.Context, info *RelayInfo, request ImageRequest) (any, error)
    ConvertOpenAIResponsesRequest(c *gin.Context, info *RelayInfo, request OpenAIResponsesRequest) (any, error)
    DoRequest(c *gin.Context, info *RelayInfo, requestBody io.Reader) (any, error)
    DoResponse(c *gin.Context, resp *http.Response, info *RelayInfo) (usage any, err *NewAPIError)
    GetModelList() []string
    GetChannelName() string
    ConvertClaudeRequest(...) (any, error)
    ConvertGeminiRequest(...) (any, error)
}
```

**TaskAdaptor 接口**:
```go
type TaskAdaptor interface {
    Init(info *RelayInfo)
    ValidateRequestAndSetAction(c *gin.Context, info *RelayInfo) *TaskError
    EstimateBilling(c *gin.Context, info *RelayInfo) map[string]float64
    AdjustBillingOnSubmit(info *RelayInfo, taskData []byte) map[string]float64
    AdjustBillingOnComplete(task *Task, taskResult *TaskInfo) int
    BuildRequestURL(info *RelayInfo) (string, error)
    BuildRequestHeader(c *gin.Context, req *http.Request, info *RelayInfo) error
    BuildRequestBody(c *gin.Context, info *RelayInfo) (io.Reader, error)
    DoRequest(c *gin.Context, info *RelayInfo, requestBody io.Reader) (*http.Response, error)
    DoResponse(c *gin.Context, resp *http.Response, info *RelayInfo) (taskID string, taskData []byte, err *TaskError)
    FetchTask(baseUrl, key string, body map[string]any, proxy string) (*http.Response, error)
    ParseTaskResult(respBody []byte) (*TaskInfo, error)
}
```

**中继处理器**:

| 文件 | 职责 |
|---|---|
| [compatible_handler.go](file:///workspace/relay/compatible_handler.go) | **核心**: 聊天补全、文本补全的统一处理（参数覆盖、Token估算、流式处理） |
| [claude_handler.go](file:///workspace/relay/claude_handler.go) | Claude Messages API 处理 |
| [gemini_handler.go](file:///workspace/relay/gemini_handler.go) | Gemini Native API 处理 |
| [image_handler.go](file:///workspace/relay/image_handler.go) | 图像生成处理 |
| [embedding_handler.go](file:///workspace/relay/embedding_handler.go) | 文本嵌入处理 |
| [audio_handler.go](file:///workspace/relay/audio_handler.go) | 音频转写/合成处理 |
| [rerank_handler.go](file:///workspace/relay/rerank_handler.go) | 重排序处理 |
| [mjproxy_handler.go](file:///workspace/relay/mjproxy_handler.go) | Midjourney 代理处理 |
| [relay_task.go](file:///workspace/relay/relay_task.go) | 异步任务提交 |
| [responses_handler.go](file:///workspace/relay/responses_handler.go) | Responses API 处理 |
| [websocket.go](file:///workspace/relay/websocket.go) | WebSocket 中继 |

**RelayInfo 上下文** ([relay/common/relay_info.go](file:///workspace/relay/common/relay_info.go)):

中继信息是多态的中继操作信息对象，包含请求元信息、渠道信息的函数处理器。

---

### 4.7 服务层 (service/)

服务层封装业务逻辑，供控制器和中间件调用。

| 文件 | 职责 |
|---|---|
| [channel.go](file:///workspace/service/channel.go) | 渠道业务逻辑 |
| [channel_select.go](file:///workspace/service/channel_select.go) | 渠道选择（随机、加权、重试） |
| [channel_affinity.go](file:///workspace/service/channel_affinity.go) | 渠道亲和性缓存（最近成功渠道优先） |
| [convert.go](file:///workspace/service/convert.go) | OpenAI ↔ Claude 格式转换 |
| [pre_consume_quota.go](file:///workspace/service/pre_consume_quota.go) | 预扣配额 |
| [quota.go](file:///workspace/service/quota.go) | 配额管理 |
| [billing.go](file:///workspace/service/billing.go) | 计费结算 |
| [tiered_settle.go](file:///workspace/service/tiered_settle.go) | 阶梯计费结算 |
| [token_counter.go](file:///workspace/service/token_counter.go) | Token 计数（tiktoken） |
| [token_estimator.go](file:///workspace/service/token_estimator.go) | Token 估算 |
| [tokenizer.go](file:///workspace/service/tokenizer.go) | Token 编码器初始化 |
| [text_quota.go](file:///workspace/service/text_quota.go) | 文本配额计算 |
| [task.go](file:///workspace/service/task.go) | 异步任务服务 |
| [task_billing.go](file:///workspace/service/task_billing.go) | 任务计费 |
| [task_polling.go](file:///workspace/service/task_polling.go) | 任务轮询 |
| [midjourney.go](file:///workspace/service/midjourney.go) | Midjourney 服务 |
| [image.go](file:///workspace/service/image.go) | 图像处理服务 |
| [audio.go](file:///workspace/service/audio.go) | 音频处理服务 |
| [download.go](file:///workspace/service/download.go) | 文件下载 |
| [http_client.go](file:///workspace/service/http_client.go) | HTTP 客户端初始化 |
| [str.go](file:///workspace/service/str.go) | 字符串工具 |
| [error.go](file:///workspace/service/error.go) | 错误处理 |
| [subscription_reset_task.go](file:///workspace/service/subscription_reset_task.go) | 订阅重置定时任务 |
| [notify-limit.go](file:///workspace/service/notify-limit.go) | 额度通知 |
| [user_notify.go](file:///workspace/service/user_notify.go) | 用户通知 |
| [ranking.go](file:///workspace/service/ranking.go) | 排名服务 |
| [file_service.go](file:///workspace/service/file_service.go) | 文件上传服务 |
| [file_decoder.go](file:///workspace/service/file_decoder.go) | 文件解码 |
| [openai_chat_responses_compat.go](file:///workspace/service/openai_chat_responses_compat.go) | Chat ↔ Responses 兼容 |
| [openai_chat_responses_mode.go](file:///workspace/service/openai_chat_responses_mode.go) | Responses 模式判断 |
| [epay.go](file:///workspace/service/epay.go) | 易支付集成 |
| [waffo_pancake.go](file:///workspace/service/waffo_pancake.go) | Waffo Pancake 支付集成 |
| [webhook.go](file:///workspace/service/webhook.go) | Webhook 回调 |
| [funding_source.go](file:///workspace/service/funding_source.go) | 资金源管理 |
| [codex_oauth.go](file:///workspace/service/codex_oauth.go) | Codex OAuth 流程 |
| [codex_credential_refresh.go](file:///workspace/service/codex_credential_refresh.go) | Codex 凭证刷新 |
| [sensitive.go](file:///workspace/service/sensitive.go) | 敏感词检测 |

**渠道选择策略** ([service/channel_select.go](file:///workspace/service/channel_select.go)):

1. **亲和性优先**: 最近成功使用的渠道 + 模型组合被优先匹配
2. **随机选择**: 从可用渠道中按权重/优先级随机选取
3. **自动重试**: 失败后自动重试下一个渠道
4. **跨组重试**: 支持配置跨分组重试

---

### 4.8 公共模块 (common/)

公共模块提供跨模块共享的基础功能。

| 文件 | 职责 |
|---|---|
| [constants.go](file:///workspace/common/constants.go) | 全局常量与变量（版本、角色、限流参数等） |
| [env.go](file:///workspace/common/env.go) | 环境变量读取与初始化 |
| [init.go](file:///workspace/common/init.go) | 初始化配置（端口、速率限制等） |
| [redis.go](file:///workspace/common/redis.go) | Redis 客户端初始化与工具函数 |
| [database.go](file:///workspace/common/database.go) | 数据库类型常量和配置 |
| [crypto.go](file:///workspace/common/crypto.go) | 密码哈希、AES 加密 |
| [gin.go](file:///workspace/common/gin.go) | Gin 上下文工具函数 |
| [utils.go](file:///workspace/common/utils.go) | 通用工具函数 |
| [email.go](file:///workspace/common/email.go) | 邮件发送（SMTP） |
| [email-outlook-auth.go](file:///workspace/common/email-outlook-auth.go) | Outlook OAuth2 邮件发送 |
| [verification.go](file:///workspace/common/verification.go) | 验证码生成与校验 |
| [totp.go](file:///workspace/common/totp.go) | TOTP 两步验证 |
| [json.go](file:///workspace/common/json.go) | JSON 工具函数 |
| [model.go](file:///workspace/common/model.go) | 模型名正则匹配 |
| [rate-limit.go](file:///workspace/common/rate-limit.go) | 通用限流器 |
| [limiter/limiter.go](file:///workspace/common/limiter/limiter.go) | Redis Lua 脚本限流器 |
| [copy.go](file:///workspace/common/copy.go) | 深拷贝工具 |
| [custom-event.go](file:///workspace/common/custom-event.go) | 事件触发 |
| [disk_cache.go](file:///workspace/common/disk_cache.go) | 磁盘缓存 |
| [sys_log.go](file:///workspace/common/sys_log.go) | 系统日志 |
| [system_monitor.go](file:///workspace/common/system_monitor.go) | 系统资源监控 |
| [pprof.go](file:///workspace/common/pprof.go) | pprof 性能分析配置 |
| [pyro.go](file:///workspace/common/pyro.go) | Pyroscope 持续分析 |
| [validate.go](file:///workspace/common/validate.go) | 输入验证 |
| [str.go](file:///workspace/common/str.go) | 字符串工具 |
| [hash.go](file:///workspace/common/hash.go) | 哈希工具 |
| [ip.go](file:///workspace/common/ip.go) | IP 地址工具 |
| [page_info.go](file:///workspace/common/page_info.go) | 分页工具 |
| [quota.go](file:///workspace/common/quota.go) | 配额单位工具 |
| [topup-ratio.go](file:///workspace/common/topup-ratio.go) | 充值比例工具 |
| [ssrf_protection.go](file:///workspace/common/ssrf_protection.go) | SSRF 防护 |
| [url_validator.go](file:///workspace/common/url_validator.go) | URL 验证 |
| [body_storage.go](file:///workspace/common/body_storage.go) | 请求体存储（可重复读取） |
| [go-channel.go](file:///workspace/common/go-channel.go) | Go Channel 封装 |
| [gopool.go](file:///workspace/common/gopool.go) | 协程池封装 |
| [endpoint_type.go](file:///workspace/common/endpoint_type.go) | 端点类型 |
| [performance_config.go](file:///workspace/common/performance_config.go) | 性能配置 |

---

### 4.9 常量模块 (constant/)

| 文件 | 职责 |
|---|---|
| [channel.go](file:///workspace/constant/channel.go) | 渠道类型常量（57 种渠道）、默认 BaseURL、名称映射 |
| [context_key.go](file:///workspace/constant/context_key.go) | Gin Context Key 常量（Token/Channel/User 相关） |
| [api_type.go](file:///workspace/constant/api_type.go) | API 类型映射 |
| [setup.go](file:///workspace/constant/setup.go) | 初始化状态相关 |
| [task.go](file:///workspace/constant/task.go) | 任务平台常量 |
| [finish_reason.go](file:///workspace/constant/finish_reason.go) | 完成原因常量 |
| [azure.go](file:///workspace/constant/azure.go) | Azure 相关常量 |
| [midjourney.go](file:///workspace/constant/midjourney.go) | Midjourney 相关常量 |
| [cache_key.go](file:///workspace/constant/cache_key.go) | 缓存 Key 前缀 |
| [env.go](file:///workspace/constant/env.go) | 环境变量名常量 |

**渠道类型 ID 映射**:

| ID | 名称 | ID | 名称 | ID | 名称 |
|---|---|---|---|---|---|
| 1 | OpenAI | 15 | Baidu | 35 | MiniMax |
| 2 | Midjourney | 16 | Zhipu | 37 | Dify |
| 3 | Azure | 17 | Ali | 40 | SiliconFlow |
| 4 | Ollama | 18 | Xunfei | 41 | VertexAI |
| 14 | Anthropic | 23 | Tencent | 42 | Mistral |
| 24 | Gemini | 25 | Moonshot | 43 | DeepSeek |
| 33 | AWS | 34 | Cohere | 48 | xAI |
| 39 | Cloudflare | 38 | Jina | 57 | Codex |

---

### 4.10 DTO 层 (dto/)

数据传输对象，定义 API 请求和响应的数据结构。

| 文件 | 职责 |
|---|---|
| [openai_request.go](file:///workspace/dto/openai_request.go) | OpenAI 请求结构（Chat/Completion） |
| [openai_response.go](file:///workspace/dto/openai_response.go) | OpenAI 响应结构 |
| [openai_image.go](file:///workspace/dto/openai_image.go) | 图像生成请求/响应 |
| [openai_video.go](file:///workspace/dto/openai_video.go) | 视频生成请求/响应 |
| [claude.go](file:///workspace/dto/claude.go) | Claude 请求/响应结构 |
| [gemini.go](file:///workspace/dto/gemini.go) | Gemini 请求/响应结构 |
| [embedding.go](file:///workspace/dto/embedding.go) | 嵌入请求/响应 |
| [audio.go](file:///workspace/dto/audio.go) | 音频请求/响应 |
| [rerank.go](file:///workspace/dto/rerank.go) | 重排序请求/响应 |
| [realtime.go](file:///workspace/dto/realtime.go) | 实时 API 结构 |
| [midjourney.go](file:///workspace/dto/midjourney.go) | Midjourney 请求/响应 |
| [suno.go](file:///workspace/dto/suno.go) | Suno 请求/响应 |
| [task.go](file:///workspace/dto/task.go) | 异步任务结构 |
| [error.go](file:///workspace/dto/error.go) | 错误响应结构 |
| [notify.go](file:///workspace/dto/notify.go) | 通知结构 |
| [pricing.go](file:///workspace/dto/pricing.go) | 定价查询结构 |
| [sensitive.go](file:///workspace/dto/sensitive.go) | 敏感词检测结构 |
| [request_common.go](file:///workspace/dto/request_common.go) | 公共请求字段 |
| [values.go](file:///workspace/dto/values.go) | 值定义 |
| [user_settings.go](file:///workspace/dto/user_settings.go) | 用户设置结构 |
| [openai_compaction.go](file:///workspace/dto/openai_compaction.go) | 响应压缩请求 |
| [playground.go](file:///workspace/dto/playground.go) | Playground 请求 |
| [channel_settings.go](file:///workspace/dto/channel_settings.go) | 渠道设置结构 |

---

### 4.11 OAuth 模块 (oauth/)

| 文件 | 职责 |
|---|---|
| [registry.go](file:///workspace/oauth/registry.go) | OAuth 提供商注册表 |
| [provider.go](file:///workspace/oauth/provider.go) | Provider 工厂接口 |
| [types.go](file:///workspace/oauth/types.go) | 公共类型定义 |
| [generic.go](file:///workspace/oauth/generic.go) | 通用 OAuth 提供商实现 |
| [github.go](file:///workspace/oauth/github.go) | GitHub OAuth |
| [discord.go](file:///workspace/oauth/discord.go) | Discord OAuth |
| [linuxdo.go](file:///workspace/oauth/linuxdo.go) | LinuxDO OAuth |
| [oidc.go](file:///workspace/oauth/oidc.go) | 通用 OIDC 提供商 |

支持的 OAuth 登录方式：GitHub、Discord、OIDC、LinuxDO、微信、Telegram。

---

### 4.12 设置模块 (setting/)

运行时配置项的读取和管理。

| 文件 | 职责 |
|---|---|
| [setting/config/config.go](file:///workspace/setting/config/config.go) | 配置结构定义和加载 |
| [ratio_setting/](file:///workspace/setting/ratio_setting/) | 模型倍率设置（缓存、暴露、分组倍率） |
| [model_setting/](file:///workspace/setting/model_setting/) | 模型特定设置（Claude/Gemini/Grok/Qwen） |
| [operation_setting/](file:///workspace/setting/operation_setting/) | 运营设置（通用、支付、配额、Token等） |
| [billing_setting/](file:///workspace/setting/billing_setting/) | 阶梯计费设置 |
| [console_setting/](file:///workspace/setting/console_setting/) | 控制台配置 |
| [system_setting/](file:///workspace/setting/system_setting/) | 系统设置（Discord/OIDC/Passkey/法律条款） |
| [performance_setting/](file:///workspace/setting/performance_setting/) | 性能配置 |
| [perf_metrics_setting/](file:///workspace/setting/perf_metrics_setting/) | 性能指标配置 |
| [reasoning/](file:///workspace/setting/reasoning/) | 推理模型后缀处理 |
| [auto_group.go](file:///workspace/setting/auto_group.go) | 自动分组配置 |
| [chat.go](file:///workspace/setting/chat.go) | 聊天设置 |
| [midjourney.go](file:///workspace/setting/midjourney.go) | Midjourney 设置 |
| [rate_limit.go](file:///workspace/setting/rate_limit.go) | 限流设置 |
| [sensitive.go](file:///workspace/setting/sensitive.go) | 敏感词设置 |
| [user_usable_group.go](file:///workspace/setting/user_usable_group.go) | 用户可用分组 |
| [payment_stripe.go](file:///workspace/setting/payment_stripe.go) | Stripe 支付设置 |
| [payment_creem.go](file:///workspace/setting/payment_creem.go) | Creem 支付设置 |
| [payment_waffo.go](file:///workspace/setting/payment_waffo.go) | Waffo 支付设置 |

---

### 4.13 工具包 (pkg/)

独立的工具包，与其他模块低耦合。

**billingexpr/ - 计费表达式引擎**:
- 编译和执行计费表达式
- 支持阶乘定价和四舍五入
- 用于渠道计费结算的灵活计算

**cachex/ - 缓存抽象层**:
- 混合缓存（内存 + Redis）
- 支持编解码器和命名空间

**ionet/ - IO.NET 模型部署客户端**:
- IO.NET API 客户端
- 管理模型容器和部署
- 硬件类型查询

**perf_metrics/ - 性能指标**:
- 请求延迟、Token 消耗量等性能指标收集
- 定时 flush 到数据库

---

### 4.14 日志模块 (logger/)

**文件**: [logger/logger.go](file:///workspace/logger/logger.go)

- 基于 Go 标准库 log 封装的统一日志器
- 支持 Debug 级别日志（需启用 `DEBUG` 环境变量）
- 提供 `LogWarn`, `LogError`, `LogInfo` 等分级方法

---

### 4.15 国际化模块 (i18n/)

| 文件 | 职责 |
|---|---|
| [i18n.go](file:///workspace/i18n/i18n.go) | i18n 初始化、翻译函数、语言检测 |
| [keys.go](file:///workspace/i18n/keys.go) | 翻译 key 常量 |
| [locales/en.yaml](file:///workspace/i18n/locales/en.yaml) | 英文翻译 |
| [locales/zh-CN.yaml](file:///workspace/i18n/locales/zh-CN.yaml) | 简体中文翻译 |
| [locales/zh-TW.yaml](file:///workspace/i18n/locales/zh-TW.yaml) | 繁体中文翻译 |

翻译查找顺序：Accept-Language 头 → Cookie → 默认英文(EN)。

---

### 4.16 前端 (web/)

项目包含两套前端，通过主题设置切换。

#### 默认前端 (web/default/)

- **技术栈**: React 19 + TypeScript + TanStack Router + shadcn/ui + TailwindCSS + Rsbuild
- **构建产物**: `web/default/dist/`
- **包管理**: Bun

**核心目录**:
| 目录 | 职责 |
|---|---|
| `src/components/` | 通用 UI 组件（搜索、对话框、加载状态等） |
| `src/hooks/` | 自定义 Hooks（认证、通知、侧边栏、媒体查询等） |
| `src/lib/` | 工具库（API 客户端、格式化、主题、缓存等） |

**关键文件**:
| 文件 | 职责 |
|---|---|
| [main.tsx](file:///workspace/web/default/src/main.tsx) | 应用入口 |
| [routeTree.gen.ts](file:///workspace/web/default/src/routeTree.gen.ts) | 路由树（自动生成） |
| [rsbuild.config.ts](file:///workspace/web/default/rsbuild.config.ts) | 构建配置 |

#### 经典前端 (web/classic/)

- **技术栈**: React 18 + JavaScript + Semi UI + Vite
- **构建产物**: `web/classic/dist/`

**核心目录**:
| 目录 | 职责 |
|---|---|
| `src/constants/` | 常量配置 |
| `src/contexts/` | React Context |
| `src/helpers/` | 辅助函数（API、认证、渲染等） |
| `src/i18n/` | 前端国际化 |
| `src/services/` | 前端服务 |

---

## 5. 数据模型 ER 关系

```
User (1) ───────────< (n) Token
User (1) ───────────< (n) Log
User (1) ───────────< (n) Midjourney
User (1) ───────────< (n) TopUp
User (1) ───────────< (n) Task
User (1) ───────────< (n) Redemption (通过 used_user_id)
User (1) ───────────< (n) Checkin
User (1) ───────────< (n) TwoFA
User (1) ───────────< (n) PasskeyCredential
User (1) ───────────< (n) UserSubscription
User (1) ───────────< (n) UserOAuthBinding

Token (1) ──────────< (n) Log

Channel (1) ────────< (n) Ability
Channel (1) ────────< (n) Log

SubscriptionPlan (1) ──< (n) SubscriptionOrder
SubscriptionPlan (1) ──< (n) UserSubscription

Vendor (1) ─────────< (n) Model

CustomOAuthProvider (1) ──< (n) UserOAuthBinding
```

**状态机**:
- `User.Status`: Enabled(1) ↔ Disabled(2)
- `Token.Status`: Enabled(1) / Disabled(2) / Expired(3) / Exhausted(4)
- `Channel.Status`: Enabled(1) / ManuallyDisabled(2) / AutoDisabled(3)
- `TopUp.Status`: pending → success / failed / expired
- `Task.Status`: pending → running → success / failure

---

## 6. 依赖关系

### Go 后端依赖

| 类别 | 核心依赖 | 用途 |
|---|---|---|
| Web 框架 | gin-gonic/gin | HTTP 框架 |
| ORM | gorm.io/gorm + gorm.io/driver/* | 数据库 ORM |
| 数据库 | glebarez/sqlite | SQLite 驱动 |
| Redis | go-redis/redis/v8 | Redis 客户端 |
| 认证 | golang-jwt/jwt, go-webauthn/webauthn | JWT + WebAuthn |
| 会话 | gin-contrib/sessions | 服务端会话管理 |
| 验证 | go-playground/validator | 请求参数验证 |
| OTP | pquerna/otp | TOTP 两步验证 |
| JSON 快速解析 | tidwall/gjson + tidwall/sjson | 高性能 JSON 处理 |
| Token 计数 | tiktoken-go/tokenizer | 精确 Token 计数 |
| 云计算 | aws-sdk-go-v2 | AWS Bedrock |
| 音视频处理 | go-audio/*, yapingcat/gomedia | 音频格式处理 |
| 支付 | stripe-go, go-epay, waffo-go | 多样化支付集成 |
| 并发 | bytedance/gopkg, samber/lo | 协程池、函数式工具 |
| 性能 | grafana/pyroscope-go | 持续性能分析 |
| 图片处理 | golang.org/x/image | 图片编解码 |
| SSH/代理 | gorilla/websocket | WebSocket |
| 数据复制 | jinzhu/copier | 结构体深拷贝 |
| 环境变量 | joho/godotenv | .env 文件加载 |
| 其他 | google/uuid, shopspring/decimal | UUID、精确小数 |

### 前端依赖

**默认前端**:
- `@tanstack/react-router` + `@tanstack/react-query` - 路由和数据获取
- `@tanstack/react-table` - 表格组件
- `@visactor/react-vchart` - 图表
- `@hugeicons/*` - 图标库
- `@lobehub/icons` - AI 供应商图标
- `@base-ui/react` - 无头 UI 组件
- `@tailwindcss/postcss` - Tailwind CSS v4
- `rsbuild` - 构建工具

**经典前端**:
- `@douyinfe/semi-ui` - Semi Design UI 组件库
- `react-router-dom` - 路由
- `vite` + `@vitejs/plugin-react` - 构建工具
- `i18next` - 国际化
- `sse.js` - 流式传输

---

## 7. 项目运行方式

### 7.1 Docker Compose 部署（推荐）

```bash
# 生产部署
docker compose up -d

# 默认端口: 3000
# 访问: http://localhost:3000
```

默认使用 PostgreSQL + Redis。可通过修改 `docker-compose.yml` 切换到 MySQL。

### 7.2 Docker 构建

```bash
# 构建镜像
docker build -t new-api .

# 运行
docker run -d -p 3000:3000 \
  -e SQL_DSN="postgresql://user:pass@host:5432/db" \
  -e REDIS_CONN_STRING="redis://:pass@host:6379" \
  new-api
```

### 7.3 Makefile 开发模式

```bash
# 构建两个前端
make build-all-frontends

# 启动后端
make start-backend

# 构建默认前端
make build-frontend

# 构建经典前端
make build-frontend-classic

# Docker 开发环境
make dev-api          # 启动后端服务
make dev-web          # 启动默认前端开发服务器
make dev-web-classic  # 启动经典前端开发服务器
make dev              # 全部启动

# 重置初始化状态
make reset-setup
```

### 7.4 本地开发

**后端**:
```bash
# 复制环境变量
cp .env.example .env

# 编辑 .env 配置数据库连接

# 直接运行
go run main.go

# 编译
go build -o new-api

# 运行编译产物
./new-api
```

**默认前端**:
```bash
cd web/default
bun install
bun run dev       # 开发服务器
bun run build     # 生产构建
bun run lint      # 代码检查
bun run typecheck # 类型检查
```

**经典前端**:
```bash
cd web/classic
bun install
bun run dev       # 开发服务器
bun run build     # 生产构建
```

### 7.5 关键环境变量

| 变量 | 说明 | 默认值 |
|---|---|---|
| `PORT` | 服务端口 | 3000 |
| `SQL_DSN` | 主数据库连接串 | (SQLite) |
| `LOG_SQL_DSN` | 日志数据库连接串 | (同主库) |
| `REDIS_CONN_STRING` | Redis 连接串 | (不使用) |
| `SESSION_SECRET` | 会话密钥 | (随机生成) |
| `SYNC_FREQUENCY` | 缓存同步间隔(秒) | 60 |
| `STREAMING_TIMEOUT` | 流式超时(秒) | 120 |
| `RELAY_TIMEOUT` | 中继超时(秒) | 0(不限制) |
| `DEBUG` | 调试模式 | false |
| `GIN_MODE` | Gin 模式 | release |
| `NODE_TYPE` | 节点类型(master) | (主节点) |
| `CHANNEL_UPDATE_FREQUENCY` | 渠道更新频率(秒) | 0(关闭) |
| `BATCH_UPDATE_ENABLED` | 批量更新 | false |
| `ENABLE_PPROF` | pprof 分析 | false |
| `MEMORY_CACHE_ENABLED` | 内存缓存 | (Redis启用则自动启用) |
| `FRONTEND_BASE_URL` | 前后端分离URL | (内置前端) |

### 7.6 CI/CD

| 工作流 | 文件 | 触发条件 |
|---|---|---|
| PR Check | [pr-check.yml](file:///workspace/.github/workflows/pr-check.yml) | Pull Request |
| Docker Build | [docker-build.yml](file:///workspace/.github/workflows/docker-build.yml) | Push 到 main |
| Docker Alpha | [docker-image-alpha.yml](file:///workspace/.github/workflows/docker-image-alpha.yml) | Alpha 分支 |
| Docker Nightly | [docker-image-nightly.yml](file:///workspace/.github/workflows/docker-image-nightly.yml) | 每日构建 |
| Electron Build | [electron-build.yml](file:///workspace/.github/workflows/electron-build.yml) | 手动触发 |
| Release | [release.yml](file:///workspace/.github/workflows/release.yml) | Tag 发布 |
| Sync to Gitee | [sync-to-gitee.yml](file:///workspace/.github/workflows/sync-to-gitee.yml) | 定时同步 |

---

## 8. API 路由清单

### 8.1 公开 API（无需认证）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/setup` | 获取初始化状态 |
| POST | `/api/setup` | 执行初始化 |
| GET | `/api/status` | 服务状态 |
| GET | `/api/uptime/status` | Uptime Kuma 状态 |
| GET | `/api/notice` | 系统通知 |
| GET | `/api/user-agreement` | 用户协议 |
| GET | `/api/privacy-policy` | 隐私政策 |
| GET | `/api/about` | 关于信息 |
| GET | `/api/home_page_content` | 首页内容 |
| POST | `/api/user/register` | 用户注册 |
| POST | `/api/user/login` | 用户登录 |
| POST | `/api/stripe/webhook` | Stripe 回调 |
| POST | `/api/creem/webhook` | Creem 回调 |
| POST | `/api/waffo/webhook` | Waffo 回调 |
| POST | `/api/waffo-pancake/webhook/:env` | Waffo Pancake 回调 |
| GET/POST | `/api/user/epay/notify` | 易支付回调 |
| GET/POST | `/api/subscription/epay/notify` | 订阅易支付回调 |

### 8.2 OAuth 登录路由

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/oauth/state` | 生成 OAuth 状态码 |
| POST | `/api/oauth/email/bind` | 邮箱绑定 |
| GET | `/api/oauth/wechat` | 微信登录 |
| GET | `/api/oauth/telegram/login` | Telegram 登录 |
| GET | `/api/oauth/telegram/bind` | Telegram 绑定 |
| GET | `/api/oauth/:provider` | 标准 OAuth（GitHub/Discord/OIDC/LinuxDO） |

### 8.3 用户 API（需认证）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/user/self` | 获取个人信息 |
| PUT | `/api/user/self` | 更新个人信息 |
| DELETE | `/api/user/self` | 删除账户 |
| GET | `/api/user/token` | 生成访问令牌 |
| GET | `/api/user/aff` | 获取推广码 |
| POST | `/api/user/topup` | 充值 |
| POST | `/api/user/pay` | 余额支付 |
| POST | `/api/user/checkin` | 签到 |
| PUT | `/api/user/setting` | 更新用户设置 |

### 8.4 管理 API（Admin）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/channel/` | 获取所有渠道 |
| POST | `/api/channel/` | 创建渠道 |
| PUT | `/api/channel/` | 更新渠道 |
| DELETE | `/api/channel/:id` | 删除渠道 |
| GET | `/api/channel/test` | 测试所有渠道 |
| POST | `/api/channel/fix` | 修复渠道能力 |
| GET | `/api/token/` | 获取所有令牌 |
| POST | `/api/token/` | 创建令牌 |
| PUT | `/api/token/` | 更新令牌 |
| DELETE | `/api/token/:id` | 删除令牌 |
| GET | `/api/user/` | 获取所有用户 |
| POST | `/api/user/` | 创建用户 |
| PUT | `/api/user/` | 更新用户 |
| DELETE | `/api/user/:id` | 删除用户 |
| GET | `/api/log/` | 获取使用日志 |
| DELETE | `/api/log/` | 删除历史日志 |
| GET | `/api/data/` | 获取配额数据 |
| GET | `/api/redemption/` | 获取兑换码 |
| POST | `/api/redemption/` | 创建兑换码 |
| GET | `/api/models/` | 获取模型元数据 |
| POST | `/api/models/` | 创建模型元数据 |
| GET | `/api/mj/` | 获取 Midjourney 任务 |
| GET | `/api/task/` | 获取异步任务 |

### 8.5 Root API

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/option/` | 获取系统选项 |
| PUT | `/api/option/` | 更新系统选项 |
| POST | `/api/option/payment_compliance` | 支付合规确认 |
| GET | `/api/performance/stats` | 性能统计 |
| DELETE | `/api/performance/disk_cache` | 清理磁盘缓存 |
| POST | `/api/performance/reset_stats` | 重置性能统计 |
| POST | `/api/performance/gc` | 强制 GC |
| POST | `/api/ratio_sync/fetch` | 同步上游倍率 |

### 8.6 中继 API（Token 认证）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/v1/chat/completions` | 聊天补全 |
| POST | `/v1/completions` | 文本补全 |
| POST | `/v1/responses` | Responses API |
| POST | `/v1/responses/compact` | 压缩响应 |
| POST | `/v1/images/generations` | 图像生成 |
| POST | `/v1/images/edits` | 图像编辑 |
| POST | `/v1/embeddings` | 文本嵌入 |
| POST | `/v1/audio/transcriptions` | 语音转写 |
| POST | `/v1/audio/translations` | 语音翻译 |
| POST | `/v1/audio/speech` | 语音合成 |
| POST | `/v1/rerank` | 重排序 |
| POST | `/v1/moderations` | 内容审核 |
| POST | `/v1/messages` | Claude Messages |
| GET | `/v1/models` | 模型列表 |
| GET | `/v1/models/:model` | 模型详情 |
| GET | `/v1/realtime` | WebSocket 实时 API |
| POST | `/v1beta/models/*path` | Gemini Native API |
| POST | `/mj/submit/*` | Midjourney 提交 |
| GET | `/mj/task/:id/fetch` | Midjourney 任务查询 |
| POST | `/suno/submit/:action` | Suno 提交 |
| POST | `/suno/fetch` | Suno 任务查询 |
| GET | `/suno/fetch/:id` | Suno 按 ID 查询 |
| POST | `/v1/videos` | 视频生成提交 |
| GET | `/v1/videos/:id` | 视频任务查询 |

---

> **最后更新**: 基于仓库最新代码分析生成 | **维护者**: New API 社区