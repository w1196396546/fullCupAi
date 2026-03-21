# Concerns Map

基于仓库内源码、部署文件与现有测试文件的定点扫描，下面只记录**有证据支撑**的风险点、技术债与脆弱区域。

## Critical

### 1. 内置的 Antigravity OAuth `client_secret` 属于硬编码敏感凭据

- 证据：`backend/internal/pkg/antigravity/oauth.go:24-29` 定义了固定 `ClientID` 与环境变量名。
- 证据：`backend/internal/pkg/antigravity/oauth.go:55-56` 直接内置 `defaultClientSecret = "GOCSPX-..."`。
- 证据：`backend/internal/pkg/antigravity/oauth.go:63-65` 只有在环境变量存在时才覆盖默认值，说明默认 secret 会直接参与运行。
- 风险：仓库泄露即等于该 OAuth 客户端密钥泄露；轮换困难；多个环境会共享同一凭据；审计时也很难区分“显式配置”与“默认兜底”。

## High

### 2. 默认部署链路依赖 `AUTO_SETUP=true`，并在首次启动时自动生成 JWT secret 与管理员密码

- 证据：`deploy/docker-compose.yml:32-38`、`deploy/docker-compose.dev.yml:21-28`、`deploy/docker-compose.standalone.yml:32` 默认开启 `AUTO_SETUP=true`。
- 证据：`deploy/README.md:111-123` 明确写明首次启动会“生成 JWT secret”“创建管理员账号（如未提供密码则自动生成）”“写入 config.yaml”。
- 证据：`backend/internal/setup/setup.go:282-289` 在未提供 JWT secret 时自动生成随机 secret。
- 证据：`backend/internal/setup/setup.go:389-395` 在未提供管理员密码时自动生成密码，并直接打印到标准输出。
- 证据：`deploy/README.md:123-125` 建议通过容器日志检索生成出的管理员密码。
- 风险：容器重建、环境变量遗漏或多实例部署时，认证密钥与初始管理员凭据会变成“运行时副产物”，增加运维漂移与日志泄漏面。

### 3. 组用量汇总存在全表聚合瓶颈，数据量上来后容易拖慢后台统计

- 证据：`backend/internal/repository/usage_log_repo.go:3137-3149` 注释明确写明“该查询会扫描 ALL usage_logs rows for total_cost aggregation”。
- 证据：`backend/internal/repository/usage_log_repo.go:3141-3152` 通过 `LEFT JOIN usage_logs` + `SUM(...)` 对全量日志做累计与当日汇总，没有看到预聚合或缓存。
- 风险：`usage_logs` 过百万行后，管理后台统计接口容易放大数据库 CPU、I/O 与锁竞争。

## Medium

### 4. 管理与运维路径存在多处“接口已暴露、逻辑未完成”的占位实现

- 证据：`backend/internal/service/proxy_service.go:172-181` 的 `TestConnection` 只取出代理记录后直接 `return nil`，实际探测逻辑仍是 TODO。
- 证据：`backend/internal/service/account_service.go:380-397` 的 `TestCredentials` 对 Anthropic/OpenAI/Gemini 三个平台均是 TODO 后直接返回 `nil`。
- 证据：`backend/internal/service/admin_service.go:1849-1857` 的 `RefreshAccountCredentials` 只读取账号并返回，刷新逻辑未实现。
- 证据：`backend/internal/handler/admin/group_handler.go:361-369` 的分组统计接口当前直接返回全 0 mock 数据，并注明 `TODO: implement actual stats`。
- 风险：这类占位接口最脆弱的地方不只是“功能缺失”，而是调用方会误以为操作成功，导致后台排障与运营判断失真。

### 5. 与上述高风险链路相邻的测试保护不足，尤其是安装流程与分组用量汇总

- 证据：`backend/internal/setup/setup_test.go:1-78` 当前只覆盖 `decideAdminBootstrap`、`setupDefaultAdminConcurrency`、`writeConfigFile`，未看到 `Install`、`createAdminUser`、`AUTO_SETUP` 链路的行为测试。
- 证据：测试代码中仅看到一个 stub 形式的 `GetAllGroupUsageSummary`：`backend/internal/handler/sora_gateway_handler_test.go:351`。
- 证据：真实逻辑入口在 `backend/internal/service/dashboard_service.go:172-175` 与 `backend/internal/handler/admin/group_handler.go:373-385`，但本次仓库搜索未命中对应 handler/service 层测试调用。
- 风险：部署初始化、管理员引导、分组成本汇总这几条链路一旦回归，问题更可能在集成环境或线上才暴露。

### 6. 若干超大文件已成为高耦合热点，后续改动的连带影响面偏大

- 证据：`backend/internal/repository/usage_log_repo.go` 约 4237 行。
- 证据：`backend/internal/service/admin_service.go` 约 2682 行。
- 证据：`backend/internal/handler/gateway_handler.go` 约 1745 行。
- 证据：`backend/internal/handler/admin/setting_handler.go` 约 1608 行。
- 证据：上述热点中还叠加了功能 TODO 与统计/配置类核心逻辑，例如 `backend/internal/repository/usage_log_repo.go:3139`、`backend/internal/service/admin_service.go:1855`。
- 风险：文件粒度过大时，单次改动很难做局部心智建模，review 成本、回归半径和冲突概率都会升高。

## Short-Term Priorities

1. 立即移除 `backend/internal/pkg/antigravity/oauth.go` 中的硬编码 `client_secret`，改成“未配置即失败”的显式启动策略。
2. 收紧默认部署面：为 `AUTO_SETUP` 增加更强的生产保护条件，并停止把自动生成的管理员密码打印到日志。
3. 优先治理 `backend/internal/repository/usage_log_repo.go` 的分组用量汇总路径，至少补缓存或预聚合。
4. 把占位实现补齐或直接下线相关入口，优先顺序建议为：`account_service.go`、`proxy_service.go`、`admin_service.go`、`group_handler.go`。
5. 为安装流程与分组用量汇总补回归测试，避免“默认配置 + 首次启动”场景继续裸奔。
