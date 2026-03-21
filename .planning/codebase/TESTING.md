# Sub2API 测试模式速记

这份文档只记录接手仓库时最需要先知道的测试事实，结论基于仓库当前真实文件，而不是理想状态。

## 1. 测试框架全景

### 后端 `backend`

- 主框架是 Go 自带 `testing`。
- 断言库以 `github.com/stretchr/testify` 为主，`require` 最常见，`assert` 和 `suite` 次之。
  - 依据：`backend/go.mod`，以及 `backend/internal/pkg/response/response_test.go`、`backend/internal/repository/concurrency_cache_integration_test.go`。
- HTTP/路由测试大量使用 `net/http/httptest` 和 `gin.TestMode`。
  - 依据：`backend/internal/server/middleware/recovery_test.go`、`backend/internal/testutil/httptest.go`。
- 集成测试使用 `testcontainers-go` 拉起 PostgreSQL / Redis。
  - 依据：`backend/internal/repository/integration_harness_test.go`、`backend/internal/middleware/rate_limiter_integration_test.go`。
- 少量数据库单测使用 `go-sqlmock`。
  - 依据：`backend/go.mod`，以及 `rg` 统计结果显示有 5 个测试文件使用 `sqlmock`。
- 仓库内还有基准测试，命名为 `BenchmarkXxx`。
  - 当前共 16 个测试文件含 benchmark。

### 前端 `frontend`

- 使用 `Vitest + Vue Test Utils + jsdom`。
  - 依据：`frontend/package.json`、`frontend/vitest.config.ts`。
- 覆盖率使用 V8 provider，阈值配置为全局 80%。
  - 依据：`frontend/vitest.config.ts`。

## 2. 测试目录分布

### 后端分布

- 测试基本与源码同目录共置，`backend/internal/**` 下遍布 `_test.go`。
- 当前 `backend` 一共有 383 个 `_test.go` 文件。
- 共享单元测试工具集中在 `backend/internal/testutil`：
  - `fixtures.go`
  - `httptest.go`
  - `stubs.go`
- 集成测试核心底座在 `backend/internal/repository/integration_harness_test.go`。
- E2E 测试集中在 `backend/internal/integration`。

### 构建标签分布

- `//go:build unit`：125 个后端测试文件。
- `//go:build integration`：32 个后端测试文件。
- `//go:build e2e`：3 个后端测试文件。
- 未打标签：223 个后端测试文件。
- 这意味着标签使用并不完全统一；未打标签的测试会被普通 `go test ./...` 和 `go test -tags=unit ./...` 一起跑到。

### 前端分布

- Vitest 文件主要在 `frontend/src/**/__tests__`。
- 当前 `frontend/src` 下有 50 个 `*.spec.*` / `*.test.*` 文件。
- 目录覆盖 `api`、`components`、`composables`、`router`、`stores`、`utils`，还有 `src/__tests__/integration`。

## 3. 运行命令

### 根目录

- `make test`
  - 实际执行：后端 `make -C backend test`，前端 `pnpm --dir frontend run lint:check` + `pnpm --dir frontend run typecheck`
  - 注意：**不会**运行前端 Vitest。
- `make test-backend`
- `make test-frontend`

### 后端 `backend/Makefile`

- `make test`
  - 执行 `go test ./...`
  - 然后执行 `golangci-lint run ./...`
- `make test-unit`
  - 执行 `go test -tags=unit ./...`
- `make test-integration`
  - 执行 `go test -tags=integration ./...`
- `make test-e2e`
  - 目标声明为 `./scripts/e2e-test.sh`
  - 但当前仓库里 **不存在** `backend/scripts/e2e-test.sh`
- `make test-e2e-local`
  - 执行 `go test -tags=e2e -v -timeout=300s ./internal/integration/...`

### 前端

- `pnpm --dir frontend run test`
- `pnpm --dir frontend run test:run`
- `pnpm --dir frontend run test:coverage`
- `pnpm --dir frontend run lint:check`
- `pnpm --dir frontend run typecheck`

## 4. mock / fixture / harness 模式

### 后端单元测试

- 最常见模式是在同一个 `_test.go` 文件里定义局部 `stubXxx` / `fakeXxx` / `mockXxx`。
  - 依据：`backend/internal/handler/sora_gateway_handler_test.go`、`backend/internal/service/openai_gateway_record_usage_test.go`。
- 共享 fixture 放在 `backend/internal/testutil/fixtures.go`，通过 `NewTestUser(...)`、`NewTestAccount(...)`、`NewTestAPIKey(...)`、`NewTestGroup(...)` 生成默认对象，再用 opts 覆盖。
- 共享 Gin 测试上下文放在 `backend/internal/testutil/httptest.go` 的 `NewGinTestContext(...)`。
- 共享空实现 stub 放在 `backend/internal/testutil/stubs.go`，并带有编译期接口断言。
- 常见测试辅助 API：
  - `t.Cleanup(...)`
  - `t.Setenv(...)`
  - `t.TempDir()`
  - `t.Parallel()`

### 后端集成测试

- `backend/internal/repository/integration_harness_test.go` 用 `TestMain` 统一拉起 PostgreSQL 与 Redis 容器，并初始化全局 `ent.Client` / `redis.Client`。
- 数据库隔离主要靠事务回滚辅助：
  - `testTx(...)`
  - `testEntTx(...)`
- Redis 隔离通过 key 前缀命名空间和 cleanup 完成。
- 数据准备辅助函数集中在 `backend/internal/repository/fixtures_integration_test.go`，命名为 `mustCreateUser(...)`、`mustCreateGroup(...)`、`mustCreateAccount(...)` 等。
- 一部分集成测试使用 `testify/suite` 组织，例如 `ConcurrencyCacheSuite`。

### 后端 E2E

- E2E 构建标签是 `e2e`，目录在 `backend/internal/integration`。
- `backend/internal/integration/e2e_helpers_test.go` 支持 `E2E_MOCK=true` 的 mock 模式。
- 若既没有真实 API Key、又没开 mock 模式，测试会显式 `t.Skip(...)`。
- E2E 日志会做简单脱敏，例如 `safeLogKey(...)` 只显示前 8 位。

### 前端测试

- 前端常通过 `vi.mock(...)` 在模块级替换 API、i18n、router 等依赖。
  - 依据：`frontend/src/api/__tests__/client.spec.ts`、`frontend/src/components/__tests__/LoginForm.spec.ts`。
- 全局浏览器环境补丁与 mock 放在 `frontend/src/__tests__/setup.ts`，包括 `requestIdleCallback`、`IntersectionObserver`、`ResizeObserver`。
- 组件测试使用 `mount(...)` / `flushPromises()`；状态管理测试配合 `Pinia`。

## 5. 现状缺口与接手注意点

- `backend/Makefile` 的 `test-e2e` 依赖 `./scripts/e2e-test.sh`，但当前仓库没有这个脚本。
- 根目录 `make test` 不会执行前端 Vitest；如果你只跑根命令，会漏掉 `frontend/src/**/__tests__`。
- 后端测试标签没有完全统一，223 个 `_test.go` 没有 `unit/integration/e2e` 标签；筛选测试范围时要注意“无标签测试仍会参与普通测试”。
- 集成测试对 Docker 的依赖并不完全同一策略：
  - `backend/internal/repository/integration_harness_test.go` 在 CI 缺 Docker 时直接失败。
  - `backend/internal/middleware/rate_limiter_integration_test.go` 在本地缺 Docker 时会跳过。
- 前端 Vitest 已配置覆盖率阈值，但根目录默认测试流程没有把 `test:coverage` 接进去。

## 6. 一句话理解现有测试体系

后端是“同目录 `_test.go` + testify + httptest + testcontainers + 少量 suite/benchmark”的主力体系，前端是“Vitest + Vue Test Utils + jsdom”的补充体系；真正需要警惕的是命令入口不统一，以及 `test-e2e` 脚本声明与仓库现状不一致。
