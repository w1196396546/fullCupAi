# Sub2API 架构测绘

> 目标：帮助后续规划快速回答三个问题。
> 1. 服务从哪里启动
> 2. 请求如何穿过路由、处理器、业务层、存储层
> 3. 哪些目录是稳定边界，哪些目录是能力堆叠点

## 1. 一句话结论

`sub2api` 的主运行体是一个 **Go + Gin 的单体 API Gateway / Admin Backend**，通过 `Google Wire` 做依赖装配，用 `Ent + PostgreSQL` 作为主存储、`Redis` 作为缓存和协调层，并可把 `Vue 3` 前端静态资源嵌入后端二进制统一对外提供。

---

## 2. 启动入口

### 2.1 主入口

- 运行入口在 `backend/cmd/server/main.go`
- `main()` 先初始化 bootstrap logger，再分三种模式：
  - `-version`：打印版本号
  - `-setup`：走 CLI 安装流程，入口在 `backend/internal/setup`
  - 默认：检测是否需要安装，然后进入 setup server 或 main server

### 2.2 Setup 启动链

- `backend/cmd/server/main.go`
  - `setup.NeedsSetup()` 决定是否首次安装
  - 未安装时调用 `runSetupServer()`
- `runSetupServer()` 直接创建一个最小 Gin 实例
  - 注册 setup 路由：`backend/internal/setup/handler.go`
  - 挂载安全相关中间件：`backend/internal/server/middleware`
  - 如果存在嵌入前端，则通过 `backend/internal/web` 对外提供安装向导页面

### 2.3 正常启动链

```text
`backend/cmd/server/main.go`
  -> `config.LoadForBootstrap()`
  -> `initializeApplication(...)` in `backend/cmd/server/wire.go`
  -> Wire 组装 config / repository / service / middleware / handler / server
  -> `app.Server.ListenAndServe()`
```

### 2.4 HTTP Server 装配

- `backend/internal/server/http.go`
  - `ProvideRouter(...)` 生成 `*gin.Engine`
  - `ProvideHTTPServer(...)` 生成 `*http.Server`
  - 这里统一处理：
    - Gin mode
    - trusted proxies
    - 全局请求体限制
    - H2C / HTTP2 cleartext
    - header 与 idle timeout

---

## 3. 请求流

## 3.1 管理后台 / 用户 API 请求流

```text
HTTP Request
  -> `backend/internal/server/router.go`
  -> `backend/internal/server/routes/*.go`
  -> `backend/internal/server/middleware/*.go`
  -> `backend/internal/handler/**/*.go`
  -> `backend/internal/service/*.go`
  -> `backend/internal/repository/*.go`
  -> PostgreSQL / Redis / 外部 HTTP 服务
```

关键分组：

- 通用探活：`/health`，注册于 `backend/internal/server/routes/common.go`
- 用户与认证 API：`/api/v1/auth`、`/api/v1/user`、`/api/v1/keys` 等，注册于 `backend/internal/server/routes/auth.go`、`backend/internal/server/routes/user.go`
- 管理 API：`/api/v1/admin/...`，注册于 `backend/internal/server/routes/admin.go`
- 网关 API：`/v1`、`/v1beta`、`/responses`、`/chat/completions`、`/antigravity/...`、`/sora/...`，注册于 `backend/internal/server/routes/gateway.go`

## 3.2 Gateway 主链路

以 Claude/OpenAI/Gemini 兼容网关为例：

```text
Client
  -> route group in `backend/internal/server/routes/gateway.go`
  -> API Key 鉴权 / 订阅校验 / 请求体限制 / 请求ID / Ops 错误记录 / 平台强制中间件
  -> `backend/internal/handler/gateway_handler.go`
  -> `backend/internal/service/gateway_service.go`
  -> 账号调度 / 速率限制 / 并发控制 / sticky session / 计费 / usage 记录
  -> `backend/internal/repository/account_repo.go` 等仓储
  -> Redis 缓存 + PostgreSQL 持久化 + 上游模型 HTTP 请求
```

从 `backend/internal/handler/gateway_handler.go` 可以看到 Handler 本身不承担复杂业务判定，而是负责：

- 从 context 取出鉴权结果
- 解析和校验请求体
- 设置请求上下文元数据
- 协调并发槽位和等待队列
- 调用 `GatewayService`、`BillingCacheService`、`UsageService`
- 将 service 返回错误翻译成协议兼容响应

这说明网关入口采用的是 **薄 Handler + 厚 Service**。

## 3.3 Setup 请求流

```text
Browser
  -> `/setup/*`
  -> `backend/internal/setup/handler.go`
  -> `backend/internal/setup/setup.go`
  -> 连接测试 / 写配置 / 安装锁
```

`setupGuard()` 明确把“安装阶段接口”与“正常运行阶段接口”隔离开，防止重复安装。

## 3.4 前端资源流

- 前端入口：`frontend/src/main.ts`
- 路由入口：`frontend/src/router/index.ts`
- 构建产物在 Docker 构建阶段被复制到 `backend/internal/web/dist`
- 运行期由 `backend/internal/web/embed_on.go` 注入 `window.__APP_CONFIG__` 后再返回 `index.html`

这意味着部署形态默认是 **后端统一承载 API + 前端静态资源**，不是前后端分离强依赖。

---

## 4. 分层与职责

## 4.1 Server / Route 层

相关路径：

- `backend/internal/server/http.go`
- `backend/internal/server/router.go`
- `backend/internal/server/routes`
- `backend/internal/server/middleware`

职责：

- 创建 Gin/HTTP Server
- 注册 route group
- 装配通用中间件
- 做协议入口分流

这里的边界比较干净：`server` 只知道 handler 和 middleware，不直接碰数据库。

## 4.2 Handler 层

相关路径：

- `backend/internal/handler`
- `backend/internal/handler/admin`
- `backend/internal/handler/dto`
- 聚合入口：`backend/internal/handler/wire.go`

职责：

- 处理 HTTP 参数绑定、基础校验、响应编码
- 把协议差异转换为统一 service 输入
- 对外暴露按业务分组的 handler 集合

显著特征：

- `admin` 子目录把后台管理接口单独隔离
- `dto` 专门放请求/响应映射，避免 service 类型直接暴露到 API 边界

## 4.3 Service 层

相关路径：

- `backend/internal/service`

职责：

- 核心业务规则
- 鉴权、计费、调度、速率限制、并发控制、幂等、订阅、公告、备份、更新、运维监控
- 组合多个 repository/cache/external client

这是项目最重的层，也是后续规划最容易继续膨胀的目录。

几个关键服务簇：

- Gateway 簇：`gateway_service.go`、`openai_gateway_service.go`、`gemini_messages_compat_service.go`、`antigravity_gateway_service.go`
- Auth / User 簇：`auth_service.go`、`api_key_service.go`、`subscription_service.go`
- 调度与约束簇：`concurrency_service.go`、`rate limit` 相关文件、`scheduler_snapshot_service.go`
- 运维与统计簇：`ops_*`、`dashboard_*`
- 后台任务簇：`*_cleanup_service.go`、`token_refresh_service`、`scheduled_test_runner_service`

## 4.4 Repository / Storage 层

相关路径：

- `backend/internal/repository`
- Ent schema：`backend/ent/schema`
- SQL migrations：`backend/migrations`

职责：

- PostgreSQL 数据访问
- Redis 缓存与分布式协调
- 外部 HTTP 客户端适配
- 备份对象存储、GitHub Release、Turnstile、OAuth client 等基础设施能力

从 `backend/internal/repository/wire.go` 能看出 repository 层实际承载了三类东西：

- DB Repository
- Redis Cache
- External Client Adapter

也就是说，这里的 “repository” 更接近 **Infrastructure Layer**，而不只是传统数据库仓储。

---

## 5. 服务层 / 存储层职责拆分

## 5.1 配置与启动基础设施

- 配置模型与加载：`backend/internal/config/config.go`
- Wire 配置提供：`backend/internal/config/wire.go`
- DB 初始化与迁移：`backend/internal/repository/ent.go`
- Redis 初始化：`backend/internal/repository/redis.go`

关键点：

- `LoadForBootstrap()` 允许在首次启动时容忍缺失的 `jwt.secret`
- `InitEnt()` 内部会执行 migrations 并补齐 bootstrap secrets
- `RunMode` 支持 `standard` 与 `simple`

## 5.2 认证与用户侧

- Handler：`backend/internal/handler/auth_handler.go`、`user_handler.go`、`api_key_handler.go`
- Service：`backend/internal/service/auth_service.go`、`api_key_service.go`、`subscription_service.go`
- Repository：`user_repo.go`、`api_key_repo.go`、`user_subscription_repo.go`、`refresh_token_cache.go`

职责拆分清晰：

- handler 关心 HTTP 行为
- service 关心注册策略、邀请码、邮件验证、token 策略
- repository 关心用户、Key、订阅的读写和缓存

## 5.3 API Gateway / 调度 / 计费

- Handler：`backend/internal/handler/gateway_handler.go`、`openai_gateway_handler.go`、`sora_gateway_handler.go`
- Service：
  - `backend/internal/service/gateway_service.go`
  - `backend/internal/service/openai_gateway_service.go`
  - `backend/internal/service/billing_service.go`
  - `backend/internal/service/billing_cache_service.go`
  - `backend/internal/service/concurrency_service.go`
  - `backend/internal/service/scheduler_snapshot_service.go`
- Repository：
  - `backend/internal/repository/account_repo.go`
  - `backend/internal/repository/usage_log_repo.go`
  - `backend/internal/repository/usage_billing_repo.go`
  - `backend/internal/repository/gateway_cache.go`
  - `backend/internal/repository/concurrency_cache.go`
  - `backend/internal/repository/scheduler_cache.go`

这是系统最关键的数据流：

```text
请求解析
  -> 鉴权拿到 API Key / User / Subscription
  -> 校验余额与资格
  -> 选择 group / platform / account
  -> 应用 sticky session / 速率限制 / 并发控制
  -> 转发到上游模型
  -> 记录 usage / 计费 / 统计 / 错误事件
```

## 5.4 运维、统计与后台任务

- 管理路由：`backend/internal/server/routes/admin.go`
- Admin handlers：`backend/internal/handler/admin`
- Ops services：`backend/internal/service/ops_*`
- Ops repository：`backend/internal/repository/ops_repo*.go`

特点：

- 实时信号、告警、错误重试、系统日志、Dashboard 全部在同一条纵向链路里
- 统计表与聚合逻辑已经较重，说明系统不只是网关，还在向“可观测运营后台”发展

---

## 6. 关键依赖方向

主依赖方向基本是单向的：

```text
`cmd/server`
  -> `internal/config`
  -> `internal/server`
  -> `internal/handler`
  -> `internal/service`
  -> `internal/repository`
  -> `ent` / `migrations` / PostgreSQL / Redis / external APIs
```

前端独立方向：

```text
`frontend/src`
  -> build output
  -> `backend/internal/web/dist`
  -> `backend/internal/web/embed_on.go`
  -> browser
```

比较稳定的边界：

- `server` 不直接依赖 repository
- `handler` 不直接依赖 ent/sql
- `repository` 通过接口回流给 service

相对容易继续耦合的位置：

- `backend/internal/service`
- `backend/internal/repository/wire.go`
- `backend/cmd/server/wire_gen.go`

因为它们承担了大量组合式依赖。

---

## 7. 主要模式

## 7.1 Dependency Injection

- 明确使用 `Google Wire`
- 入口文件：`backend/cmd/server/wire.go`
- 生成结果：`backend/cmd/server/wire_gen.go`

价值：

- 启动图显式
- 替换依赖时可测试
- 也暴露出当前对象图已经很大

## 7.2 Repository Pattern

- 典型实现见 `backend/internal/repository/account_repo.go`
- 业务层依赖接口，基础设施层提供实现

## 7.3 Thin Handler / Rich Service

- 典型实现见 `backend/internal/handler/gateway_handler.go`
- Handler 做协议胶水，Service 承担主要业务规则

## 7.4 Embedded Frontend

- Docker 构建时把前端 `dist` 拷入 `backend/internal/web/dist`
- 运行时通过 `backend/internal/web/embed_on.go` 注入配置并返回页面

## 7.5 Migration-as-Source-of-Truth

- `backend/internal/repository/ent.go` 明确优先执行 SQL migration
- `backend/migrations/*.sql` 是 schema 的真实演进轨迹
- `backend/ent/schema` 更像 ORM 模型声明和代码生成来源

## 7.6 Cache + Snapshot + Outbox

从以下路径能看出明显的缓存快照化趋势：

- `backend/internal/repository/scheduler_cache.go`
- `backend/internal/repository/scheduler_outbox_repo.go`
- `backend/internal/service/scheduler_snapshot_service.go`

这说明账号调度不是每次都直接打 DB，而是逐步演进为：

```text
DB 变更
  -> outbox
  -> snapshot rebuild
  -> Redis/cache
  -> 请求读快照
```

这对高并发网关是很关键的架构方向。

---

## 8. 规划时最值得关注的抽象

### 8.1 `Handlers`

- 聚合定义在 `backend/internal/handler/wire.go`
- 是 HTTP 能力总装配点

### 8.2 `GatewayService`

- 主文件：`backend/internal/service/gateway_service.go`
- 是平台兼容、调度、会话粘性、计费联动的核心

### 8.3 `repository.ProviderSet`

- 文件：`backend/internal/repository/wire.go`
- 是“所有基础设施能力”注入清单

### 8.4 `SetupRouter` / `Register*Routes`

- `backend/internal/server/router.go`
- `backend/internal/server/routes/*.go`

这是新人最快建立 HTTP 地图的位置。

---

## 9. 后续规划建议

如果后面要继续做 codebase mapping、拆分改造或性能治理，优先顺序建议是：

1. 从 `backend/cmd/server/main.go`、`backend/cmd/server/wire.go` 建立完整启动图
2. 从 `backend/internal/server/routes/gateway.go` + `backend/internal/handler/gateway_handler.go` 追网关热路径
3. 从 `backend/internal/service/gateway_service.go`、`openai_gateway_service.go`、`scheduler_snapshot_service.go` 识别核心调度抽象
4. 从 `backend/internal/repository/wire.go` 和 `backend/migrations` 识别基础设施边界与数据模型演进

---

## 10. 本文依据的关键文件

- `backend/cmd/server/main.go`
- `backend/cmd/server/wire.go`
- `backend/internal/server/http.go`
- `backend/internal/server/router.go`
- `backend/internal/server/routes/admin.go`
- `backend/internal/server/routes/auth.go`
- `backend/internal/server/routes/gateway.go`
- `backend/internal/server/routes/user.go`
- `backend/internal/handler/wire.go`
- `backend/internal/handler/gateway_handler.go`
- `backend/internal/service/gateway_service.go`
- `backend/internal/service/auth_service.go`
- `backend/internal/repository/wire.go`
- `backend/internal/repository/ent.go`
- `backend/internal/repository/redis.go`
- `backend/internal/repository/account_repo.go`
- `backend/internal/config/config.go`
- `backend/internal/setup/handler.go`
- `backend/internal/setup/setup.go`
- `backend/internal/web/embed_on.go`
- `frontend/src/main.ts`
- `frontend/src/router/index.ts`
- `Dockerfile`
