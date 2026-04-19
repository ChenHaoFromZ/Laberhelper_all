# 改动清单 — 前端 & 中间件 vs 原仓库

> 生成时间：2026-04-19  
> 基准分支：`laborlawhelp` → `origin/OpenHarness-based`；`laborlawhelp-middlend` → `origin/main`

---

## 一、前端（`laborlawhelp`）

### 总改动量
14 个文件，+701 行 / -83 行（不含新增文件）

### 1.1 核心功能文件

| 文件 | 改动方向 | 关键内容 |
|------|----------|----------|
| `src/app/consultation/page.tsx` | +520 行 | ★ 主要功能升级（见下） |
| `src/features/consultation/services/middleware-api.ts` | +63 行 | 新增 `listSessionMessages` API；补充 capability 声明 |
| `src/hooks/use-case-store.tsx` | +45 行 | 新增 `replaceMessages`、`SessionReference` 类型导出 |

#### `consultation/page.tsx` 详细改动

- **新增辅助函数**
  - `normalizeReferences()` — 规范化来自 SSE 的引用条目（citations）
  - `shortenId()` — 截断展示 case/session ID
  - `humanizeToolName()` — 将工具名称人性化（如 `mcp__pkulaw__get_article` → `PKULaw · get_article`）
  - `createAnonymousOwnerToken()` — 生成匿名用户 token（使用 `crypto.randomUUID`）

- **新增常量**
  - `MIDDLEWARE_CLIENT_CAPABILITIES`：声明前端支持 `['citations', 'tool-status', 'structured-final', 'trace-id']`
  - `CONSULTATION_SESSION_STORAGE_KEY`：会话 ID 持久化 key

- **新增 UI 能力**
  - 展示 AI 回复时的 tool-status（工具调用进度，如"正在检索 PKULaw…"）
  - 展示 structured-final 结构化最终答复
  - 展示 citations 法律依据引用列表（带 `ArrowUpRight` 外链图标）
  - `LoaderCircle` 动画 loading 状态
  - 会话历史消息恢复：页面重载后通过 `listSessionMessages` 拉取历史记录

- **环境变量**
  - 新增读取 `NEXT_PUBLIC_MIDDLEWARE_POLICY_VERSION` / `NEXT_PUBLIC_POLICY_VERSION`

### 1.2 脚本（`scripts/`）

| 文件 | 改动 |
|------|------|
| `build.mjs` / `build.sh` | 统一入口脚本，对齐新端口/环境变量命名 |
| `dev.mjs` / `dev.sh` | 同上（开发模式） |
| `prepare.sh` | 新增依赖检查逻辑 |

### 1.3 文档与配置

| 文件 | 改动 |
|------|------|
| `.env.example` | 新增 1 行（`NEXT_PUBLIC_MIDDLEWARE_POLICY_VERSION`） |
| `README.md` | 更新项目说明，+24 行 |
| `docs/architecture.md` | 架构图更新 |
| `docs/coze-legacy-and-env.md` | 废弃说明修订 |
| `docs/laborlawhelp-frontend-integration-playbook.md` | 集成指引更新 |
| `docs/plan.md` | 开发计划更新 |

### 1.4 新增文件

| 文件 | 说明 |
|------|------|
| `docs/quick-start.md` | 新增快速上手文档 |

---

## 二、中间件（`laborlawhelp-middlend`）

### 总改动量
已追踪文件：+1521 行 / -67 行；另有 16 个新增文件/目录（未追踪）

### 2.1 核心后端文件（修改）

#### `backend/app/adapters/openharness_client.py`（+808 行）— 最大改动

- **新增运行模式：library 模式**
  - 通过懒加载（`_load_oh_modules()`）导入 `openharness.ui.runtime` 和 `openharness.engine.stream_events`
  - 支持 `oh_mode = library`：直接在进程内调用 OpenHarness 运行时，无需 HTTP 调用
- **引用提取（citations）**
  - `_walk_reference_candidates()`：递归提取工具输出中的引用（title/url/snippet）
  - `_is_pkulaw_tool()`：识别 PKULaw 工具调用
  - `_try_parse_structured_output()`：JSON / ast.literal_eval 双模式解析
  - `_truncate()`：摘要截断工具
- **重试与超时机制**
  - 读取 `settings.oh_retry_backoff_schedule` 执行指数退避
  - 按 `oh_retry_max_attempts` 控制最大重试次数
  - `oh_first_chunk_timeout_sec`：首包超时独立控制
- **SSE 协议错误阈值**：`oh_protocol_error_threshold` 限制异常行数

#### `backend/app/core/config.py`（+66 行）

新增 OpenHarness 配置字段（全部通过 `.env` 注入）：

```
oh_mode                     # mock / library / remote
oh_lib_model                # library 模式使用的模型
oh_lib_api_format           # openai / ...
oh_lib_base_url / api_key   # library 模式 API 端点
oh_lib_max_turns            # 最大对话轮数
oh_lib_cwd                  # 工作目录
oh_lib_tool_policy          # legal_minimal / full
oh_connect_timeout_sec      # 连接超时
oh_read_timeout_sec         # 读取超时
oh_first_chunk_timeout_sec  # 首包超时
oh_retry_max_attempts       # 最大重试次数
oh_retry_backoff_seconds    # 退避间隔（逗号分隔）
oh_protocol_error_threshold # 协议错误阈值
```

新增 `@model_validator`：启动时校验 URL 格式、必填字段、数值合法性。

#### `backend/app/core/store.py`（+267 行）

- 新增会话消息持久化（Redis list 结构）
- 新增 `list_session_messages()` / `append_message()` 接口
- 支持异步 case/session 管理
- 新增结构化消息存储（含 citations、tool_status 字段）

#### `backend/app/services/chat_service.py`（+226 行）

- 拆分 streaming pipeline：区分 tool-status 事件、citations 事件、structured-final 事件
- 接入 `openharness_client` 的 library 模式分支
- 新增 `audit_service` 调用（记录每次对话）
- 新增 `policy_version` 字段透传

#### `backend/app/api/v1/routes/chat.py`（+23 行）

- 接收并透传前端发送的 `client_capabilities` 列表
- 根据 capability 决定是否下发 citations / tool-status 事件

#### `backend/app/main.py`（+25 行）

- 注册新路由：`/playground`
- 挂载 `backend/app/static/` 静态目录
- 新增启动日志输出 `oh_mode`

#### `backend/app/models/schemas.py`（+5 行）

- `ChatRequest` 新增字段：`client_capabilities`、`policy_version`

### 2.2 新增文件/模块（未追踪）

| 路径 | 说明 |
|------|------|
| `backend/app/tools/__init__.py` | 本地工具包初始化 |
| `backend/app/tools/labor_compensation.py` | 劳动赔偿计算工具（N/2N/双倍工资，陕高法规则） |
| `backend/app/tools/labor_fact_extract.py` | 案情要素提取工具 |
| `backend/app/tools/labor_document.py` | 法律文书生成工具（仲裁申请书、证据清单等） |
| `backend/agent-skills/labor_pkulaw_retrieval_flow/` | PKULaw Agent Skill 定义（4 阶段工作流） |
| `backend/app/api/v1/routes/playground.py` | Playground 调试路由 |
| `backend/app/services/audit_service.py` | 对话审计服务 |
| `backend/app/static/` | 静态资源目录（供 Playground 前端使用） |
| `backend/scripts/debug_library_tool_path.py` | 调试 library 模式工具路径 |
| `backend/scripts/run_postgres_integration.sh` | Postgres 集成测试脚本 |
| `backend/sql/migrations/` | 数据库迁移脚本目录 |
| `backend/tests/integration/` | 集成测试目录 |
| `backend/tests/test_audit_service.py` | 审计服务测试 |
| `backend/tests/test_openharness_client.py` | OH 客户端单元测试 |
| `backend/tests/test_openharness_client_enrichment.py` | OH 客户端引用提取测试 |
| `backend/tests/test_playground.py` | Playground 路由测试 |
| `docs/guide/pkulaw-openharness-collaboration.md` | PKULaw × OpenHarness 协作指引 |
| `.gitignore` | 新增 `.gitignore` |

### 2.3 其他修改

| 文件 | 改动 |
|------|------|
| `backend/.env.example` | 新增 13 个 `OH_*` 配置项 |
| `backend/tests/test_smoke.py` | +113 行，覆盖 library 模式、citations、retry 场景 |
| `backend/sql/init_schema.sql` | +1 行（`audit_log` 表补充索引） |
| `frontend-sdk/stream-chat.ts` | 新增 `client_capabilities` 参数透传 |
| `docs/project/document-index.md` | 目录更新 |
| `docs/project/test-plan.md` | 测试计划补充集成测试章节 |
| `README.md` / `backend/README.md` | 自述文件更新 |

---

## 三、汇总

| 维度 | 前端 | 中间件 |
|------|------|--------|
| 修改文件数 | 14 | ~20（含已追踪） |
| 新增文件/目录 | 1 | 18 |
| 净增代码行 | +618 | +1454 |
| 核心新能力 | citations UI、tool-status 进度、会话历史恢复、匿名 token | library 模式、本地工具三件套、Playground、审计日志、重试/超时机制 |
| 配置新增 | 1 个环境变量 | 13 个 `OH_*` 环境变量 |
