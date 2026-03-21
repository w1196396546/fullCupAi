# Sub2API 编码约定速记

这份文档只总结接手仓库时最需要先知道的约定，结论来自真实代码与配置，重点覆盖 `backend`，并补充根目录与 `frontend` 的边界信息。

## 1. 先认目录分层

- 后端主入口在 `backend/cmd/server/main.go`，手工工具入口在 `backend/cmd/jwtgen/main.go`。
- 业务代码按 `internal` 分层：
  - `backend/internal/handler` 负责 HTTP 入参与出参编排。
  - `backend/internal/service` 放业务服务、端口接口、调度与后台任务。
  - `backend/internal/repository` 放数据库、Redis、HTTP 上游等基础设施实现。
  - `backend/internal/server/routes` 负责路由注册，`backend/internal/server/middleware` 负责 Gin 中间件。
  - `backend/internal/config` 负责配置加载、默认值和校验。
  - `backend/internal/pkg` 放可复用基础包，如 `logger`、`errors`、`response`。
  - `backend/internal/util` 放更窄的工具函数。
  - `backend/internal/testutil` 是单元测试共享辅助代码。
- 生成代码单独放在 `backend/ent` 和 `backend/cmd/server/wire_gen.go`。这两处文件头都标明 `DO NOT EDIT`，应改源定义并重新生成。

## 2. 命名风格

- Go 命名整体遵循驼峰，但对缩写非常明确，仓库普遍使用 `APIKey`、`OAuth`、`JWT`、`URL`、`TOTP`、`ID` 这类大写初始词。
  - 依据：`backend/.golangci.yml` 的 `staticcheck.initialisms`，以及类型名 `APIKeyHandler`、`OpenAIOAuthService`、`TOTP` 相关字段。
- 构造函数统一偏 `NewXxx`，依赖注入装配函数统一偏 `ProvideXxx`。
  - 依据：`backend/internal/service/wire.go`、`backend/internal/handler/wire.go`、`backend/cmd/server/wire_gen.go`。
- Wire provider set 统一命名为 `ProviderSet`。
  - 依据：`backend/internal/config/wire.go`、`backend/internal/handler/wire.go`。
- 测试替身命名没有单一模板，但模式很稳定：
  - 同文件局部替身常见 `stubXxx`、`fakeXxx`、`mockXxx`。
  - 共享空实现放在 `backend/internal/testutil/stubs.go`，命名为 `StubXxx`。

## 3. 目录与依赖约束

- `service` 和 `handler` 层默认不能直接依赖 `repository`、`gorm`、`redis`。
  - 依据：`backend/.golangci.yml` 的 `depguard` 规则。
- `repository` 通过实现 `service` 层接口向上提供能力，而不是反向被 `service` 引用具体实现。
  - 依据：`backend/internal/repository/user_repo.go` 返回 `service.UserRepository`，`backend/internal/service` 中大量接口定义。
- 路由、处理器、服务、仓储分工清晰：
  - `routes` 只组装路由和中间件，例如 `backend/internal/server/routes/gateway.go`。
  - `handler` 倾向薄层调用 `service`，统一出参。
  - `service` 做业务决策、调度和超时控制。
  - `repository` 负责 Ent/SQL/Redis/HTTP 细节。

## 4. 错误处理约定

- 普通错误传播以 `fmt.Errorf("...: %w", err)` 为主，仓库大量使用 `%w` 保留调用链。
  - 依据：`backend/internal/config/config.go`、`backend/internal/service/update_service.go`、`backend/internal/service/totp_service.go`。
- 需要控制 HTTP 状态码时，用 `backend/internal/pkg/errors/errors.go` 定义的 `ApplicationError`，并在 handler 中通过 `response.ErrorFrom` 输出。
  - 依据：`backend/internal/pkg/errors/errors.go`、`backend/internal/pkg/response/response.go`、`backend/internal/handler/auth_handler.go`。
- 错误判断优先 `errors.Is` / `errors.As`，尤其在启动、鉴权、上游调用和流式逻辑里。
  - 依据：`backend/cmd/server/main.go`、`backend/internal/service/auth_service.go`、`backend/internal/handler/gemini_v1beta_handler.go`。
- 启动失败或不可恢复的 CLI/进程级错误会直接 `log.Fatalf` 退出，而不是层层吞掉。
  - 依据：`backend/cmd/server/main.go`、`backend/cmd/jwtgen/main.go`。
- Handler 里优先走统一响应封装；只有协议兼容场景才直接 `c.JSON(...)` 输出特定结构。
  - 依据：`backend/internal/handler/*` 广泛使用 `response.Success` / `response.ErrorFrom`，`backend/internal/server/routes/gateway.go` 对 OpenAI/Anthropic 特殊错误体直接 `c.JSON`。

## 5. 日志约定

- 仓库有统一日志层 `backend/internal/pkg/logger`，启动阶段先 `InitBootstrap()`，加载配置后再 `Init(logger.OptionsFromConfig(...))`。
  - 依据：`backend/cmd/server/main.go`、`backend/internal/pkg/logger/logger.go`、`backend/internal/pkg/logger/config_adapter.go`。
- 请求级日志通过中间件注入 `context`，后续从上下文取 logger，而不是到处重新拼字段。
  - 依据：`backend/internal/server/middleware/request_logger.go`。
- 访问日志字段趋于结构化，常带 `component`、`request_id`、`client_request_id`、`status_code`、`latency_ms`、`path`、`method`。
  - 依据：`backend/internal/server/middleware/logger.go`。
- 兼容旧调用的日志输出仍大量使用 `logger.LegacyPrintf(...)`，新代码若要接请求上下文，更贴近现有风格的是 `logger.With(...)` 或上下文 logger。
- 日志会主动做脱敏或避免直接打印敏感值。
  - 依据：`backend/internal/pkg/response/response.go` 使用 `logredact.RedactText(...)`，`backend/internal/integration/e2e_helpers_test.go` 提供 `safeLogKey(...)`。

## 6. 配置约定

- 配置通过 Viper 读取 `config.yaml`，并允许环境变量覆盖；点号键会映射成下划线环境变量。
  - 依据：`backend/internal/config/config.go` 中 `viper.AutomaticEnv()` 与 `SetEnvKeyReplacer(strings.NewReplacer(".", "_"))`。
- 配置搜索顺序明确：
  - `DATA_DIR`
  - `/app/data`
  - 当前目录 `.`
  - `./config`
  - `/etc/sub2api`
  - 依据：`backend/internal/config/config.go:977-990` 对应逻辑。
- `LoadForBootstrap()` 允许启动早期缺少 `jwt.secret`，完整模式再做全量校验。
  - 依据：`backend/internal/config/config.go` 与 `backend/internal/config/config_test.go`。
- 加载后会做大量归一化：去空格、转小写、默认值回填、兼容旧键、弱配置警告。
  - 依据：`backend/internal/config/config.go` 的 `load(...)`。

## 7. 常见代码模式

- Gin 路由统一从 `gin.New()` 起步，再显式挂中间件；仓库里没有 `gin.Default()` 的使用。
  - 依据：`backend/cmd/server/main.go`、`backend/internal/server/routes/gateway.go`、多处测试。
- HTTP 响应统一走 `response.Success`、`response.Created`、`response.ErrorFrom`、`response.Paginated`。
  - 依据：`backend/internal/pkg/response/response.go`。
- 依赖注入靠 Wire，新增服务/仓储/handler 通常要补 `ProviderSet` 或 `ProvideXxx`。
  - 依据：`backend/cmd/server/main.go` 的 `go:generate`、`backend/internal/*/wire.go`、`backend/cmd/server/wire_gen.go`。
- 外部 IO 和清理逻辑经常显式设置 `context.WithTimeout(...)`，而不是无限等待。
  - 依据：`backend/cmd/server/main.go`、`backend/internal/service/openai_privacy_service.go`、`backend/internal/util/urlvalidator/validator.go`。
- 共享 API 协议常量、上下文键、分页和 DTO 约定会抽到专门包，而不是散落字符串。
  - 依据：`backend/internal/handler/endpoint.go`、`backend/internal/pkg/ctxkey`、`backend/internal/pkg/pagination`、`backend/internal/handler/dto`。

## 8. 接手时应避免的写法

- 不要在 `service` 或 `handler` 里直接 import `repository`、`redis`、`gorm`，这会直接撞 lint 规则。
- 不要手改 `backend/ent/**` 或 `backend/cmd/server/wire_gen.go`。
- 不要新写裸 `panic` 处理业务错误。仓库运行时代码里的 `panic` 很少，主要用于启动期不变量或测试故意触发恢复逻辑。
- 不要在常规 handler 里随意拼响应格式；优先复用 `backend/internal/pkg/response`。
- 不要跳过错误包装和错误链判断，否则会破坏现有 `errors.Is/As` 与 `response.ErrorFrom` 的配套流程。
- 不要直接在新逻辑里散落 `fmt.Print*`/`log.Print*` 作为业务日志；优先接入 `backend/internal/pkg/logger`。
- 不要把测试辅助塞进生产目录；已有共享测试辅助统一放在 `backend/internal/testutil` 或同目录 `_test.go`。

## 9. 一句话理解现有风格

这个仓库的后端风格可以概括为：`Gin 路由 + Handler 薄层 + Service 业务层 + Repository 基础设施层 + Wire 装配 + Viper 配置 + 统一错误/日志封装`，并通过 `golangci-lint` 明确守住分层边界。
