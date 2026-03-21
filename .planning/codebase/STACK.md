# 技术栈映射

## 范围

本文基于仓库内真实代码与配置整理，重点依据：
`backend/go.mod`、`backend/cmd/server/main.go`、`backend/internal/config/config.go`、
`backend/internal/server/router.go`、`backend/internal/server/routes/*.go`、
`frontend/package.json`、`frontend/vite.config.ts`、`Dockerfile`、
`deploy/docker-compose.yml`、`Makefile`、`backend/Makefile`。

## 语言与运行时

- 后端主语言是 Go，模块定义在 `backend/go.mod`，声明版本为 `go 1.26.1`。
- 前端主语言是 TypeScript + Vue SFC，`frontend/tsconfig.json` 开启 `strict: true`，目标为 `ES2020`。
- 前端开发/构建运行时依赖 Node.js + pnpm；`Dockerfile` 使用 `node:24-alpine`，前端锁文件为 `frontend/pnpm-lock.yaml`。
- 容器运行时以 Alpine 为主；`Dockerfile` 最终镜像使用 `alpine:3.21`。

## 后端核心栈

- Web 框架：`Gin`，入口在 `backend/cmd/server/main.go`，路由装配在 `backend/internal/server/router.go`。
- ORM / 数据访问：`Ent`，初始化在 `backend/internal/repository/ent.go`，schema 位于 `backend/ent/schema/`。
- 数据库驱动：`github.com/lib/pq`，当前真实主库是 PostgreSQL。
- 配置加载：`Viper`，集中定义与校验在 `backend/internal/config/config.go`。
- 依赖注入：`Google Wire`，入口 `backend/cmd/server/wire.go`，生成文件 `backend/cmd/server/wire_gen.go`。
- 缓存 / 分布式状态：`go-redis/v9`，初始化在 `backend/internal/repository/redis.go`。
- 日志：项目自有 logger 封装 + `zap`/`slog`，入口见 `backend/internal/pkg/logger/`。
- 定时任务：`robfig/cron/v3`，用于定时清理、报告、Sora 媒体清理等，见 `backend/internal/service/scheduled_test_service.go`、`backend/internal/service/sora_media_cleanup_service.go`。
- HTTP 客户端：项目统一封装在 `backend/internal/pkg/httpclient/`，外部集成大量通过该层或 `req/v3` 发起。
- WebSocket / SSE：依赖 `coder/websocket`、`gorilla/websocket`，并在网关层暴露 OpenAI Responses WS / 流式能力，见 `backend/internal/server/routes/gateway.go`。
- HTTP/2 Cleartext：主服务支持 `h2c`，见 `backend/cmd/server/main.go` 与 `backend/internal/config/config.go`。

## 前端核心栈

- 框架：`Vue 3`，应用入口在 `frontend/src/main.ts`。
- 状态管理：`Pinia`，由 `frontend/src/main.ts` 挂载。
- 路由：`vue-router`，见 `frontend/src/router/index.ts`。
- 国际化：`vue-i18n`，由 `frontend/src/main.ts` 与 `frontend/vite.config.ts` 可见。
- 构建工具：`Vite`，配置在 `frontend/vite.config.ts`。
- 样式体系：`Tailwind CSS` + `PostCSS`，配置位于 `frontend/tailwind.config.js`、`frontend/postcss.config.js`。
- 图表与数据展示：`chart.js`、`vue-chartjs`。
- 网络请求：`axios`。
- 其他明显前端依赖：`@vueuse/core`、`xlsx`、`qrcode`、`dompurify`、`driver.js`。

## 中间件与应用层能力

- 全局中间件包含请求日志、CORS、安全头 / CSP，定义在 `backend/internal/server/router.go`。
- 请求体大小限制、API Key 鉴权、JWT 鉴权、管理员鉴权、分组校验等位于 `backend/internal/server/middleware/` 与 `backend/internal/middleware/`。
- 登录 / 注册 / 发码 / 刷新 token 等接口额外叠加 Redis 限流，见 `backend/internal/server/routes/auth.go`。
- 嵌入式前端服务能力通过 `go:embed` 实现，代码位于 `backend/internal/web/embed_on.go`、`backend/internal/web/embed_off.go`。

## 对外协议 / API 兼容层

- Claude / Anthropic 兼容入口：`/v1/messages`、`/v1/models`、`/v1/usage`，见 `backend/internal/server/routes/gateway.go`。
- OpenAI 兼容入口：`/v1/chat/completions`、`/v1/responses`，另有无 `/v1` 前缀别名。
- OpenAI Responses WebSocket 入口：`GET /v1/responses` 与 `GET /responses`。
- Gemini 原生兼容入口：`/v1beta/models/*`。
- Antigravity 专用入口：`/antigravity/v1/*`、`/antigravity/v1beta/*`。
- Sora 专用入口：`/sora/v1/chat/completions` 与媒体代理 `/sora/media/*`。

## 配置入口

- 主配置结构、默认值、校验逻辑：`backend/internal/config/config.go`。
- Docker / 环境变量样例：`deploy/.env.example`。
- 完整 YAML 配置样例：`deploy/config.example.yaml`。
- 首次安装向导 / 自动初始化：`backend/internal/setup/setup.go`、`backend/internal/setup/handler.go`、`backend/internal/setup/cli.go`。
- 部署态数据目录默认是 `/app/data`，由 `backend/internal/setup/setup.go` 与 `Dockerfile` 共同体现。

## 构建与启动方式

- 后端本地构建：`backend/Makefile` 的 `build` 目标会执行 `go build -o bin/server ./cmd/server`。
- 前端本地构建：根 `Makefile` 通过 `pnpm --dir frontend run build`；前端构建产物输出到 `backend/internal/web/dist`，见 `frontend/vite.config.ts`。
- 前后端联编推荐方式：`README.md` 中给出 `frontend` 先 `pnpm install && pnpm run build`，再在 `backend` 执行 `go build -tags embed -o sub2api ./cmd/server`。
- 服务入口：`backend/cmd/server/main.go`。
- 启动参数：`-setup` 进入 CLI 安装流程，`-version` 输出版本，见 `backend/cmd/server/main.go`。
- Docker 多阶段构建：`Dockerfile` 先编译前端，再编译后端并嵌入前端，最终生成单二进制镜像。
- Compose 启动：`deploy/docker-compose.yml`、`deploy/docker-compose.local.yml`、`deploy/docker-compose.dev.yml`。
- 容器内健康检查：`Dockerfile` 与 `deploy/docker-compose.yml` 都依赖 `/health`。

## 测试与质量命令

- 后端测试 / Lint：`backend/Makefile` 中 `go test ./...` + `golangci-lint run ./...`。
- 后端分层测试：`backend/Makefile` 提供 `test-unit`、`test-integration`、`test-e2e`。
- 前端质量检查：`frontend/package.json` 提供 `lint`、`lint:check`、`typecheck`、`vitest`。
- 根 `Makefile` 的 `test` 目标会串联后端测试与前端 `lint:check`、`typecheck`。

## 结论

这是一个以 Go + Gin + Ent + Redis + PostgreSQL 为后端核心、Vue 3 + TypeScript + Vite + Tailwind 为前端核心的 AI API Gateway。其部署形态优先考虑单体服务 + 嵌入式前端 + PostgreSQL/Redis 配套组件，既支持源码构建，也支持 Docker Compose 一体化运行。
