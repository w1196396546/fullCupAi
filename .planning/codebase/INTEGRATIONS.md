# 外部集成映射

## 范围

本文只记录仓库内已出现明确代码、配置或路由入口的外部系统。主要依据：
`backend/internal/config/config.go`、`backend/internal/server/routes/*.go`、
`backend/internal/service/*.go`、`backend/internal/repository/*.go`、
`deploy/config.example.yaml`、`deploy/.env.example`、`deploy/docker-compose.yml`。

## 数据库

### PostgreSQL

- 主数据库是 PostgreSQL，初始化在 `backend/internal/repository/ent.go`。
- 安装向导会探测并在不存在时创建数据库，见 `backend/internal/setup/setup.go`。
- 迁移来源是嵌入式迁移文件系统 `migrations.FS`，在 `backend/internal/repository/ent.go` 中启动时自动执行。
- Docker Compose 默认依赖 `postgres:18-alpine`，见 `deploy/docker-compose.yml`。
- 配置入口在 `backend/internal/config/config.go` 的 `database` 段，以及 `deploy/.env.example`。

### SQLite

- `backend/go.mod` 引入了 `modernc.org/sqlite`。
- 但当前主服务初始化路径 `backend/internal/repository/ent.go` 明确只走 PostgreSQL。
- 因此在本仓库主服务范围内，SQLite 不是主运行数据库；可视为存在依赖但不是当前主链路。

## 缓存与分布式状态

### Redis

- Redis 是明确的一等依赖，初始化在 `backend/internal/repository/redis.go`。
- Docker Compose 默认依赖 `redis:8-alpine`，见 `deploy/docker-compose.yml`。
- 真实用途覆盖：
  - 认证限流：`backend/internal/middleware/rate_limiter.go`
  - 邮件验证码 / 重置 token 缓存：`backend/internal/repository/email_cache.go`
  - TOTP 缓存：`backend/internal/repository/totp_cache.go`
  - API Key / 身份 / 会话 / 订阅 / 仪表盘缓存：`backend/internal/repository/api_key_cache.go`、`identity_cache.go`、`dashboard_cache.go`
  - 并发控制、排队、会话限制：`backend/internal/repository/concurrency_cache.go`、`session_limit_cache.go`
  - 网关 / 调度 / RPM / 临时不可调度状态：`gateway_cache.go`、`scheduler_cache.go`、`rpm_cache.go`、`temp_unsched_cache.go`
  - Ops 指标与告警锁：`backend/internal/service/ops_metrics_collector.go`、`ops_alert_evaluator_service.go`、`ops_cleanup_service.go`
- 配置入口在 `backend/internal/config/config.go` 的 `redis` 段，以及 `deploy/.env.example`。

## AI 上游与第三方 API

### OpenAI

- OpenAI OAuth 常量与 token 解析在 `backend/internal/pkg/openai/oauth.go`。
- OpenAI OAuth 服务在 `backend/internal/service/openai_oauth_service.go`。
- 网关兼容 OpenAI `chat/completions` 与 `responses`，见 `backend/internal/server/routes/gateway.go`。
- OpenAI 相关默认放行域名配置在 `backend/internal/config/config.go` 与 `deploy/config.example.yaml`。

### Anthropic / Claude

- Claude OAuth 客户端在 `backend/internal/repository/claude_oauth_service.go`，直接访问 `https://claude.ai` 与 Anthropic token 端点。
- Claude OAuth 公共常量在 `backend/internal/pkg/oauth/oauth.go`。
- 网关的 `/v1/messages`、`/v1/models`、`/v1/usage` 兼容 Anthropic / Claude 协议，见 `backend/internal/server/routes/gateway.go`。
- Claude OAuth 使用量查询代码存在于 `backend/internal/repository/claude_usage_service.go`。

### Google Gemini / AI Studio / Code Assist / Antigravity

- Gemini OAuth 服务在 `backend/internal/service/gemini_oauth_service.go`。
- Gemini CLI / AI Studio 常量在 `backend/internal/pkg/geminicli/constants.go`，涉及：
  - `https://generativelanguage.googleapis.com`
  - `https://cloudcode-pa.googleapis.com`
  - Google OAuth 授权与换 token 端点
- Google Drive 配额查询存在于 `backend/internal/pkg/geminicli/drive_client.go`。
- Antigravity（Google Cloud Code Assist / Gemini 变体）客户端与 OAuth 在 `backend/internal/pkg/antigravity/client.go`、`backend/internal/pkg/antigravity/oauth.go`。
- Gemini 原生兼容接口位于 `backend/internal/server/routes/gateway.go` 的 `/v1beta/*` 与 `/antigravity/v1beta/*`。

### Sora

- Sora 客户端默认上游是 `https://sora.chatgpt.com/backend`，默认值见 `backend/internal/config/config.go`。
- Sora 网关接口见 `backend/internal/server/routes/gateway.go`。
- OpenAI/Sora 会话读取逻辑见 `backend/internal/service/openai_oauth_service.go`。
- 支持媒体代理、签名 URL 与本地 / S3 / upstream 存储标记，见 `backend/internal/service/sora_generation.go`、`backend/internal/service/sora_gateway_service.go`。

### Cloudflare Turnstile

- Turnstile 校验服务直连 `https://challenges.cloudflare.com/turnstile/v0/siteverify`，代码在 `backend/internal/repository/turnstile_service.go`。
- 前端登录 / 注册 / 忘记密码页都接入了 Turnstile 组件，见 `frontend/src/views/auth/*.vue` 与 `frontend/src/components/TurnstileWidget.vue`。
- 配置入口在 `backend/internal/config/config.go` 的 `turnstile` 段，以及 `deploy/config.example.yaml`。

### GitHub Releases

- 在线更新通过 GitHub Releases API 获取最新版本并下载文件，见 `backend/internal/repository/github_release_service.go`。
- 更新服务依赖该客户端，注入关系见 `backend/cmd/server/wire_gen.go`。
- 代理配置入口是 `update.proxy_url`，定义于 `backend/internal/config/config.go`。

### 模型价格数据源

- 价格服务会定时从远程下载模型价格 JSON 与哈希文件，代码在 `backend/internal/service/pricing_service.go`。
- 默认远程源是 `raw.githubusercontent.com` 上的 `model-price-repo` 固定 commit 文件，默认值见 `backend/internal/config/config.go`。
- URL allowlist 单独对 `pricing_hosts` 做了配置，见 `backend/internal/config/config.go` 与 `deploy/config.example.yaml`。

### CRS（外部 Relay / 账户源同步）

- 项目存在显式 CRS 同步能力，不是只留了配置。
- 服务实现位于 `backend/internal/service/crs_sync_service.go`，会登录外部 CRS、导出账户并同步到本地。
- 管理员接口入口在 `backend/internal/server/routes/admin.go`：
  - `POST /api/v1/admin/accounts/sync/crs`
  - `POST /api/v1/admin/accounts/sync/crs/preview`
- URL allowlist 专门预留了 `crs_hosts`，见 `backend/internal/config/config.go` 与 `deploy/config.example.yaml`。

## 认证与安全集成

### 本地认证 / JWT

- 本地登录态通过 JWT 实现，配置在 `backend/internal/config/config.go` 的 `jwt` 段。
- 路由鉴权在 `backend/internal/server/routes/auth.go` 与 `backend/internal/server/middleware/`。
- JWT 生成辅助命令在 `backend/cmd/jwtgen/main.go`。

### LinuxDo Connect OAuth

- 登录集成不是占位，而是有完整回调链路。
- 配置结构在 `backend/internal/config/config.go` 的 `linuxdo_connect` 段。
- 后端处理器在 `backend/internal/handler/auth_linuxdo_oauth.go`。
- 路由入口在 `backend/internal/server/routes/auth.go`：
  - `GET /api/v1/auth/oauth/linuxdo/start`
  - `GET /api/v1/auth/oauth/linuxdo/callback`
  - `POST /api/v1/auth/oauth/linuxdo/complete-registration`
- 前端回调页在 `frontend/src/views/auth/LinuxDoCallbackView.vue`。

### TOTP 双因素认证

- TOTP 作为安全能力已接入，依赖 `pquerna/otp`，业务侧见 `backend/internal/service/totp_service.go`（由 `wire_gen.go` 注入）。
- TOTP 密钥缓存使用 Redis，见 `backend/internal/repository/totp_cache.go`。

### SMTP 邮件

- 项目存在真实 SMTP 外发能力，不是文档占位。
- 发送逻辑在 `backend/internal/service/email_service.go`，直接使用 Go 标准库 `net/smtp` 与可选 TLS。
- 异步邮件队列在 `backend/internal/service/email_queue_service.go`，是进程内队列，不是外部 MQ。
- 管理员设置接口中可测试 SMTP，见 `backend/internal/server/routes/admin.go` 的 `/admin/settings/test-smtp`、`/send-test-email`。

## 对象存储与备份

### Sora S3 存储

- Sora 媒体可接入 S3 兼容对象存储；配置模型在 `backend/internal/service/settings_view.go`。
- Sora S3 配置与多 profile 管理逻辑在 `backend/internal/service/setting_service.go`。
- 管理员接口位于 `backend/internal/server/routes/admin.go` 的 `/admin/settings/sora-s3*`。

### 备份 S3 / Cloudflare R2

- 数据库备份服务存在完整上传 / 下载 / 预签名 URL / 恢复逻辑，见 `backend/internal/service/backup_service.go`。
- 该服务明确说明是 “S3 兼容存储（支持 Cloudflare R2）”。
- 注入关系在 `backend/cmd/server/wire_gen.go`，对象存储工厂为 `repository.NewS3BackupStoreFactory()`。
- 管理员路由中存在备份 S3 配置相关接口，见 `backend/internal/server/routes/admin.go` 的 backup/data-management 路由。

## 其他外部系统边界

### 外部页面嵌入（iframe）

- 项目支持将支付页、工单页或自定义页面通过 iframe 嵌入后台 / 用户端。
- 后端在 `backend/internal/server/router.go` 动态收集 frame-src origins 并注入 CSP。
- 前端页面在 `frontend/src/views/user/CustomPageView.vue`、`frontend/src/views/user/PurchaseSubscriptionView.vue`。
- 这是“嵌入外部系统”，不是项目自己实现支付或工单协议。

### 代理系统

- 多个外部 HTTP 集成支持显式代理 URL（http/https/socks5/socks5h）。
- 相关通用能力在 `backend/internal/pkg/httpclient/`、`backend/internal/pkg/proxyurl/`、`backend/internal/pkg/proxyutil/`。
- GitHub 更新、Gemini/Antigravity/OpenAI OAuth、上游网关请求都能走代理。

## 未发现

- 未发现面向第三方回调消费的业务 Webhook 接口。
- 未发现 Kafka、RabbitMQ、NATS、NSQ、AMQP 之类外部消息中间件接入。
- 未发现 gRPC 服务端或外部 gRPC 调用主链路；`grpc` 相关依赖只在依赖树中出现，当前仓库主路径未见实际接入代码。
- 未发现 Stripe / 支付宝 / 微信支付等支付 SDK 的直接接入代码；支付能力目前更像通过外部 iframe 页面集成。

## 结论

这个仓库的外部集成以 “AI 上游 + PostgreSQL/Redis + OAuth/Turnstile + SMTP + S3 兼容存储 + GitHub/价格源 + CRS 同步” 为主。Webhook 与外部 MQ 没有落地证据，支付系统也更偏向外部嵌入而不是仓库内直连实现。
