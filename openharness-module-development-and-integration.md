# OpenHarness 智能层模块开发与对接文档

## 1. 文档目标与范围
本文件用于规范 laborlawhelp-middlend 中 OpenHarness 智能层模块的设计、开发、联调与上线。

适用范围：
- 后端适配层：`backend/app/adapters/openharness_client.py`
- 聊天编排层：`backend/app/services/chat_service.py`
- SSE 出流协议层：`backend/app/core/sse.py`
- 配置层：`backend/app/core/config.py`

不包含：
- 业务规则引擎内部规则实现（仅定义对接边界）
- 前端 UI 渲染细节（见 frontend-integration 文档）

---

## 2. 上游 OpenHarness 关键能力（对接依据）
基于 OpenHarness 仓库当前公开实现，智能层对接应建立在以下稳定能力之上：

1. Agent Loop
- 采用 query -> stream -> tool_use -> tool_result -> loop 的闭环执行模型。
- 单轮可触发多个工具调用，并回传工具结果继续推理。

2. 统一流式接口语义
- API 客户端抽象使用流事件模型（text delta + message complete + retry event）。
- CLI/print 模式可输出 `stream-json`，包含 `tool_started`、`tool_completed` 等事件。

3. 工具系统与结构化输入
- 工具基于 Pydantic 输入模型，具备结构化参数校验。
- MCP 工具可被适配为统一命名格式：`mcp__<server>__<tool>`。

4. Skills/Plugins/MCP 扩展机制
- Skills 支持 Markdown 按需加载（bundled/user/plugin 三类来源）。
- 插件可携带 skill、command、hook、MCP server 配置。
- MCP 管理器支持 stdio/http/websocket 连接方式（以运行时配置为准）。

5. 会话与上下文治理
- 支持会话快照保存与恢复。
- 内置上下文压缩（session memory / full compact）及重试路径，适合长对话场景。

6. 多 Provider 兼容
- 支持 Anthropic-compatible、OpenAI-compatible、Codex、Copilot 等工作流。
- 智能层适配时应避免绑定单一厂商语义，优先保留通用字段。

---

## 3. 本项目中的智能层分层设计

## 3.1 分层职责
1. Route 层
- 接收 `POST /api/v1/sessions/{session_id}/chat/stream`
- 返回 `text/event-stream`

2. Service 层
- 会话归属校验
- 写入用户消息
- 调用 OpenHarness 适配器
- 将 OpenHarness 事件映射为前端 SSE 契约

3. Adapter 层（OpenHarnessClient）
- 提供 mock 与 remote 两种执行路径
- 屏蔽上游协议细节（SSE 行格式、事件名差异、异常形态）

4. Store 层
- 消息持久化
- 会话锁
- 流序号（seq）

## 3.2 关键调用时序
1. 客户端发起聊天请求。
2. Service 获取会话锁并保存用户消息。
3. Service 发送 `message_start`。
4. Adapter 拉取 OpenHarness 流。
5. Service 将每个 chunk 映射为 `content_delta/tool_call/tool_result/final`。
6. Service 保存助手消息并发送 `message_end`。

---

## 4. 模块接口契约

## 4.1 适配器输出契约
适配器应输出统一 chunk 模型（当前为 `OHChunk`）：
- `type`: `text | tool_call | tool_result | final`
- `content`: 文本增量（`type=text`）
- `tool_name`: 工具名称（`type=tool_call/tool_result`）
- `args`: 工具参数（`type=tool_call`）
- `metadata`: 扩展元数据（`type=tool_result/final`）

## 4.2 Service 对前端 SSE 契约映射
| 适配器 chunk.type | SSE 事件 | data 字段 |
|---|---|---|
| `text` | `content_delta` | `delta`, `seq` |
| `tool_call` | `tool_call` | `tool_name`, `args` |
| `tool_result` | `tool_result` | `tool_name`, `result_summary` |
| `final` | `final` | `message_id`, `summary`, `references`, `rule_version` |

固定补充事件：
- 流开始：`message_start`
- 流结束：`message_end`
- 异常兜底：`error`

## 4.3 上游 remote 请求建议字段
当前实现已发送：
- `prompt`
- `session_id`
- `workflow`
- `user_context`
- `output_format=stream`

建议新增（向前兼容，按需使用）：
- `trace_id`：用于链路追踪
- `locale`：用于多语言回复控制
- `policy_version`：用于规则集灰度
- `client_capabilities`：声明前端支持事件类型

---

## 5. 配置与环境规范

## 5.1 必要配置项
- `oh_base_url`
- `oh_stream_path`
- `oh_api_key`
- `oh_default_workflow`
- `oh_use_mock`

## 5.2 运行模式
1. 本地开发模式
- `oh_use_mock=true`
- 不依赖外部 OpenHarness 服务
- 用于前后端联调、回归测试

2. 集成联调/预发模式
- `oh_use_mock=false`
- 指向真实 OpenHarness 网关
- 必须开启鉴权与可观测性

## 5.3 建议的配置校验
启动时至少校验：
- `oh_use_mock=false` 时，`oh_base_url/oh_stream_path/oh_api_key` 非空
- `oh_base_url` 必须为 http/https
- `oh_stream_path` 必须以 `/` 开头

---

## 6. 错误处理与重试策略

## 6.1 错误分层
1. 传输错误
- 连接失败、超时、断流
- 映射为 `OH_SERVICE_ERROR`

2. 协议错误
- 事件名未知、data 不是合法 JSON
- 建议记录 warning 并跳过无效片段（不中断整流）

3. 业务错误
- 上游返回明确失败事件
- 透传可公开信息，隐藏内部细节

## 6.2 重试建议
- 仅对幂等阶段重试（建连/首包超时）
- 流已开始后不做自动重放（避免重复输出）
- 采用指数退避：1s、2s、4s，最多 3 次

## 6.3 错误响应结构
对前端输出 `error` 事件时建议统一：
- `code`
- `message`
- `retryable`
- `trace_id`（如有）

---

## 7. 安全与治理要求
1. 凭据安全
- `oh_api_key` 仅通过环境变量注入，不写入日志。

2. 输入安全
- `prompt` 做长度上限保护。
- `user_context` 仅传最小必要字段（例如 owner_type/owner_id）。

3. 输出安全
- 工具调用与工具结果中可能包含敏感信息，进入 SSE 前应按策略脱敏。

4. 可审计性
- 记录最小审计字段：`session_id`, `owner_id`, `workflow`, `latency_ms`, `finish_reason`。

---

## 8. 测试与验收清单

## 8.1 单元测试
1. 适配器 mock 流正确产出顺序：tool_call -> tool_result -> text* -> final。
2. remote 模式中 event/data 解析正确。
3. 非法 JSON 行被跳过且不中断。
4. HTTP >= 400 时抛出统一 `AppError`。

## 8.2 集成测试
1. `chat/stream` 可收到完整事件序列。
2. `final` 后消息成功落库。
3. 并发请求同一 session 时锁行为正确。
4. 上游故障时前端收到可重试 error 事件。

## 8.3 回归测试
- 切换 `oh_use_mock` 不影响 SSE 契约。
- 切换 `storage_backend` 不影响智能层事件序列。

---

## 9. 版本演进建议（M1/M2/M3）

### M1（当前可交付）
- mock/real 双模式打通。
- 基础事件映射稳定。
- pytest 覆盖主链路。

### M2（增强可观测）
- 增加 trace_id 全链路透传。
- 增加 token/耗时指标。
- 引入标准化错误码映射表。

### M3（生产化）
- 灰度路由（按 workflow/provider）。
- 上游多实例容错与熔断。
- 合规审计与数据留存策略完善。

---

## 10. 与现有代码的落地对照
当前代码已具备以下基础能力：
- `OpenHarnessClient` 的 mock 与 remote 双路径。
- Service 层对 `text/tool_call/tool_result/final` 的映射。
- 统一 SSE 输出与 `message_start/message_end` 包裹。

建议下一步优先补齐：
1. trace_id 透传与日志关联。
2. remote 模式超时与重试可配置化。
3. 事件解析异常计数指标。
4. 上游 finish_reason 与本地错误码映射表。
