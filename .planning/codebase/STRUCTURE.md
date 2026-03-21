# Sub2API 目录结构测绘

> 目标：让新人第一次进入仓库时，能在 5 分钟内知道“先看哪里、改哪里、别从哪里开始”。

## 1. 顶层目录总览

```text
`backend`    Go 后端主工程，真正的核心代码
`frontend`   Vue 3 管理台与用户端前端
`deploy`     Docker / systemd / 安装脚本 / 示例配置
`docs`       补充文档
`tools`      辅助脚本
`assets`     README/站点展示资源
`.planning`  规划与测绘产物
```

最重要的判断：

- **先看业务**：从 `backend` 开始
- **先看 UI**：从 `frontend/src` 开始
- **先看部署**：从 `deploy` 开始
- **先看迁移与模型**：从 `backend/migrations`、`backend/ent/schema` 开始

---

## 2. 推荐导航顺序

如果你是第一次接手这个仓库，推荐按下面顺序看：

1. `README.md`
2. `backend/cmd/server/main.go`
3. `backend/cmd/server/wire.go`
4. `backend/internal/server/router.go`
5. `backend/internal/server/routes`
6. `backend/internal/handler`
7. `backend/internal/service`
8. `backend/internal/repository`
9. `frontend/src/main.ts`
10. `frontend/src/router/index.ts`

这条路径能最快建立“入口 -> 路由 -> 处理器 -> 业务 -> 存储 -> 前端”的整体感。

---

## 3. 顶层目录详解

## 3.1 `backend`

这是项目最重要的目录，后端主程序、数据模型、迁移、测试都在这里。

关键子目录：

- `backend/cmd`
  - `backend/cmd/server`：主服务入口
  - `backend/cmd/jwtgen`：辅助 CLI 工具
- `backend/internal`
  - 应用内部代码，按分层组织
- `backend/ent`
  - Ent 生成代码和 schema
- `backend/migrations`
  - SQL migration，数据库演进事实来源
- `backend/resources`
  - 运行期静态资源，目前最关键的是模型价格数据

## 3.2 `frontend`

这是前端应用目录，使用 Vue 3 + Vite + Pinia + Vue Router。

关键子目录：

- `frontend/src/api`
  - 前端 API 封装
- `frontend/src/views`
  - 页面级视图，按 `admin` / `auth` / `setup` / `user` 分组
- `frontend/src/components`
  - 可复用 UI 组件，按业务域继续分组
- `frontend/src/stores`
  - Pinia 状态管理
- `frontend/src/router`
  - 路由定义和导航逻辑
- `frontend/src/i18n`
  - 多语言

## 3.3 `deploy`

部署与运维入口。

关键文件：

- `deploy/docker-compose.local.yml`
- `deploy/docker-compose.yml`
- `deploy/install.sh`
- `deploy/config.example.yaml`
- `deploy/docker-entrypoint.sh`

适合回答：

- 本地/服务器怎么起
- Docker 怎么配环境变量
- systemd 怎么装

## 3.4 `docs`

目前文档量不大，属于专题补充区。

- `docs/ADMIN_PAYMENT_INTEGRATION_API.md`

## 3.5 `tools`

小型工具脚本区，目前规模不大。

- `tools/check_pnpm_audit_exceptions.py`

## 3.6 `.planning`

这是规划输出目录，不属于运行时代码。

- 当前目标文件位于 ` .planning/codebase `
- 适合沉淀 architecture map、roadmap、phase plan 等非运行资产

---

## 4. `backend` 内部结构

## 4.1 `backend/cmd`

### `backend/cmd/server`

主服务启动点。

关键文件：

- `backend/cmd/server/main.go`
- `backend/cmd/server/wire.go`
- `backend/cmd/server/wire_gen.go`
- `backend/cmd/server/VERSION`

新人从这里能看清：

- 服务怎么启动
- 依赖怎么装配
- 版本信息从哪里来

### `backend/cmd/jwtgen`

辅助命令目录，不是主运行路径。

---

## 4.2 `backend/internal`

这是业务核心层，按职责拆分，结构比较稳定。

### `backend/internal/config`

职责：

- 配置结构体
- 默认值
- 环境变量/配置文件加载
- 启动期与完整期校验

先看文件：

- `backend/internal/config/config.go`
- `backend/internal/config/wire.go`

### `backend/internal/server`

职责：

- 创建 Gin / HTTP server
- 中间件装配
- route registration

关键位置：

- `backend/internal/server/http.go`
- `backend/internal/server/router.go`
- `backend/internal/server/routes`
- `backend/internal/server/middleware`

### `backend/internal/handler`

职责：

- HTTP 请求绑定与响应返回
- API 边界层
- 按接口类型分组

结构特征：

- 普通 handler 在 `backend/internal/handler`
- 后台管理 handler 在 `backend/internal/handler/admin`
- DTO 映射在 `backend/internal/handler/dto`

### `backend/internal/service`

职责：

- 业务规则与核心流程
- 跨 repository 的编排
- 调度、计费、限流、并发控制、OAuth、公告、Ops、备份等

这是仓库里最重、最需要谨慎维护的目录。

### `backend/internal/repository`

职责：

- PostgreSQL 读写
- Redis 缓存
- 外部 HTTP client
- 基础设施适配

不要把它理解得太窄。这里不是只有 DB repo，而是整个 infra adapter 集合。

### `backend/internal/setup`

职责：

- 首次安装向导
- DB / Redis 连接测试
- 写配置与安装锁

### `backend/internal/web`

职责：

- 嵌入式前端静态资源服务
- 在返回 HTML 前注入公开配置

### `backend/internal/pkg`

职责：

- 底层通用能力库

已经按主题分出较多小包，例如：

- `backend/internal/pkg/openai`
- `backend/internal/pkg/claude`
- `backend/internal/pkg/gemini`
- `backend/internal/pkg/logger`
- `backend/internal/pkg/httpclient`
- `backend/internal/pkg/response`

### `backend/internal/domain`

轻量领域常量与领域对象定义。

### `backend/internal/model`

目前规模较小，放更偏模型定义的结构。

### `backend/internal/integration`

端到端与集成测试入口，例如 `e2e_gateway_test.go`。

### `backend/internal/testutil`

测试夹具、stub、公共 helper。

### `backend/internal/util`

偏工具型、非核心业务抽象的通用辅助。

---

## 4.3 `backend/ent`

这部分是 ORM 相关代码。

重点区分：

- `backend/ent/schema`
  - schema 定义，最值得人工阅读
- `backend/ent/...`
  - 生成产物，通常不作为第一阅读入口

如果你只想看数据模型，优先看：

- `backend/ent/schema/account.go`
- `backend/ent/schema/group.go`
- `backend/ent/schema/api_key.go`
- `backend/ent/schema/usage_log.go`
- `backend/ent/schema/user.go`
- `backend/ent/schema/user_subscription.go`

---

## 4.4 `backend/migrations`

数据库真实演进轨迹。

特点：

- 文件很多，说明 schema 演进频繁
- 按编号排序执行
- 对理解历史兼容逻辑非常有帮助

重点关注：

- 早期基础表：`001_init.sql`
- 调度/统计演进：`026_ops_metrics_aggregation_tables.sql`、`034_usage_dashboard_aggregation_tables.sql`
- Sora 相关：`046_add_sora_accounts.sql`、`063_add_sora_client_tables.sql`
- 幂等与稳定性：`057_add_idempotency_records.sql`
- 调度快照相关：`036_scheduler_outbox.sql`

---

## 4.5 `backend/resources`

运行时附带资源目录。

目前最重要的是：

- `backend/resources/model-pricing/model_prices_and_context_window.json`

这和定价、模型能力、上下文窗口等逻辑强相关。

---

## 5. `frontend` 内部结构

## 5.1 启动与框架骨架

关键文件：

- `frontend/src/main.ts`
- `frontend/src/App.vue`
- `frontend/src/router/index.ts`
- `frontend/src/style.css`

理解这四个文件后，基本能建立前端全局图。

## 5.2 页面分区

`frontend/src/views` 已经按角色/阶段清晰分区：

- `frontend/src/views/setup`
  - 首次安装向导页面
- `frontend/src/views/auth`
  - 登录、注册、回调、找回密码
- `frontend/src/views/user`
  - 普通用户面板
- `frontend/src/views/admin`
  - 管理后台
- `frontend/src/views/admin/ops`
  - Ops 专项页面

这个分区和后端路由结构是对齐的，认知成本比较低。

## 5.3 组件分区

`frontend/src/components` 采用“通用 + 业务域”混合分法：

- 通用层：
  - `frontend/src/components/common`
  - `frontend/src/components/layout`
  - `frontend/src/components/icons`
  - `frontend/src/components/charts`
- 业务层：
  - `frontend/src/components/admin/...`
  - `frontend/src/components/account`
  - `frontend/src/components/keys`
  - `frontend/src/components/sora`
  - `frontend/src/components/user/...`

### 放置约定

- 页面独占组件优先放对应业务子目录
- 多页面共享组件再提升到 `common` 或 `layout`
- 图表组件单独放 `charts`

## 5.4 前端状态与 API

- 状态管理：`frontend/src/stores`
- API 调用：`frontend/src/api`

看前端问题时，经常可以沿着下面这条线查：

```text
`views`
  -> `components`
  -> `stores`
  -> `api`
  -> 后端 `/api/v1/...`
```

---

## 6. 命名与放置约定

## 6.1 后端命名

从当前仓库可以归纳出这些稳定习惯：

- `*_handler.go`：HTTP 边界
- `*_service.go`：业务逻辑
- `*_repo.go`：存储访问
- `*_cache.go`：Redis / 内存缓存适配
- `*_test.go`：单元测试
- `*_integration_test.go`：集成测试

这套命名非常直给，适合按文件名搜索。

## 6.2 路由放置

- route group 定义集中在 `backend/internal/server/routes`
- 具体业务实现不写在 route 文件里，而是交给 handler

## 6.3 管理后台隔离

后端和前端都显式做了 admin 分区：

- 后端：`backend/internal/handler/admin`、`/api/v1/admin/...`
- 前端：`frontend/src/views/admin`、`frontend/src/components/admin`

这说明“后台管理域”是一个一级模块，不是散落能力。

## 6.4 测试放置

- 后端单测通常与源码同目录
- 集成测试集中在：
  - `backend/internal/integration`
  - 带 `integration` build tag 的各 repo 测试
- 前端测试集中在：
  - `frontend/src/__tests__`
  - 各子目录 `__tests__`

---

## 7. 对新人最重要的位置导航

### 如果你要定位“服务怎么跑起来”

看：

- `backend/cmd/server/main.go`
- `backend/cmd/server/wire.go`
- `backend/internal/server/http.go`

### 如果你要定位“某个 API 从哪里进来”

看：

- `backend/internal/server/routes`
- 再跳到 `backend/internal/handler`

### 如果你要定位“业务规则写在哪”

看：

- `backend/internal/service`

### 如果你要定位“数据库字段和表”

看：

- `backend/ent/schema`
- `backend/migrations`

### 如果你要定位“前端页面入口”

看：

- `frontend/src/router/index.ts`
- `frontend/src/views`

### 如果你要定位“部署需要哪些变量”

看：

- `deploy/docker-compose.local.yml`
- `deploy/config.example.yaml`
- `deploy/README.md`

---

## 8. 最值得建立索引的目录

后续如果继续做 codebase mapping，建议优先为这些目录建立更细的二级文档：

1. `backend/internal/service`
2. `backend/internal/repository`
3. `backend/internal/handler/admin`
4. `backend/internal/server/routes`
5. `frontend/src/views/admin`

原因很简单：

- 这些目录最厚
- 变更最频繁
- 最能决定后续重构成本

---

## 9. 本文依据的关键文件

- `README.md`
- `Makefile`
- `backend/Makefile`
- `backend/cmd/server/main.go`
- `backend/cmd/server/wire.go`
- `backend/internal/server/http.go`
- `backend/internal/server/router.go`
- `backend/internal/server/routes/admin.go`
- `backend/internal/server/routes/auth.go`
- `backend/internal/server/routes/gateway.go`
- `backend/internal/server/routes/user.go`
- `backend/internal/config/config.go`
- `backend/internal/handler/wire.go`
- `backend/internal/repository/wire.go`
- `frontend/package.json`
- `frontend/src/main.ts`
- `frontend/src/router/index.ts`
- `deploy/README.md`
- `deploy/docker-compose.local.yml`
- `Dockerfile`
