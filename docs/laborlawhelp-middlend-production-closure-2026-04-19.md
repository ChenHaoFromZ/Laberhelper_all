# LaborLawHelp Middlend 生产化收口更新（2026-04-19）

## 1. 本轮已实现

### 1.1 SSE 收尾一致化
- 成功链路：`message_start -> ... -> final -> message_end`
- 失败链路：`message_start -> ... -> error -> message_end`
- `error` 字段保持：`code/message/retryable/trace_id`

对应代码：
- `laborlawhelp-middlend/backend/app/services/chat_service.py`

### 1.2 审计落库与可追溯
- 新增 `AuditService`，在 turn 成功/失败时写入 `audit_logs`
- 最小字段贯穿：`trace_id/session_id/owner_id/workflow/latency_ms/finish_reason/retry_count/error_code`
- 上游细节不透给前端，只写内部日志/审计

对应代码：
- `laborlawhelp-middlend/backend/app/services/audit_service.py`
- `laborlawhelp-middlend/backend/app/core/store.py`

### 1.3 Postgres/Redis 集成测试与脚本
- 新增集成测试：
  - `storage_backend=postgres` 成功链路
  - 同会话并发锁冲突 `SESSION_LOCKED`
  - 失败链路 `error -> message_end`
  - 失败态 assistant `messages.metadata` 可查询
  - `trace_id` 在 SSE 与 audit 一致
- 新增一键脚本：docker 启停 + schema/migration + pytest integration

对应文件：
- `laborlawhelp-middlend/backend/tests/integration/test_postgres_streaming.py`
- `laborlawhelp-middlend/backend/scripts/run_postgres_integration.sh`

### 1.4 仓库卫生与文档收口
- 新增 `.gitignore`，覆盖 `__pycache__/`、`.pytest_cache/`、`*.pyc`
- 清理当前缓存产物
- 文档索引明确单一源：
  - `laborlawhelp-middlend/docs/guide/openharness-module-development-and-integration.md`

对应文件：
- `laborlawhelp-middlend/.gitignore`
- `laborlawhelp-middlend/docs/project/document-index.md`
- `laborlawhelp-middlend/docs/project/test-plan.md`
- `laborlawhelp-middlend/backend/README.md`

## 2. 本机验证结果

- 单元/烟雾：`14 passed, 3 skipped`
- 一键集成脚本：已执行到 Docker 启动步骤，但当前用户无 Docker Socket 权限：
  - `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`

## 3. 当前已知缺口

- Postgres/Redis 实链测试代码与脚本已就绪，但本机当前会话无法直接拉起 Docker；需补充 Docker 组权限或使用 sudo 执行。

## 4. 单一文档源约定

- OpenHarness 模块开发文档以仓库内下述路径为唯一维护源：
  - `laborlawhelp-middlend/docs/guide/openharness-module-development-and-integration.md`
- 本目录（`/home/chen-hao/repositories/laborhelper/docs`）保留阶段更新与跳转说明，不再作为同主题双份正文源。
