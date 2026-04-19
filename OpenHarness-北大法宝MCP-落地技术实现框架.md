# 智裁项目技术实现框架（OpenHarness + 北大法宝 MCP + 大模型）

## 1. 文档目标

本方案用于指导“劳动争议智能分诊平台”从 0 到 1 落地，实现以下目标：

1. 建立可扩展的六层架构：交互层、业务层、规则层、模型层、数据层、治理层。
2. 以 OpenHarness 作为智能体编排框架，统一会话、工具调用、重试、压缩、持久化。
3. 通过北大法宝 MCP 提供可核查法律检索能力，降低法条/案例幻觉。
4. 构建“规则优先、模型增强、人工可复核”的法律 AI 生产线，支持西安/陕西本地化口径。

---

## 2. 建设边界与设计原则

## 2.1 一期建设边界（MVP）

1. 覆盖案型：未签合同、违法解除、工资拖欠、社保争议、基础工伤咨询。
2. 覆盖输出：案情结构卡、赔偿测算、证据目录建议、仲裁/起诉草稿、行动清单、繁简分流结果。
3. 覆盖端：Web H5 + 后台管理端（规则/模板/审核/日志）。

## 2.2 核心设计原则

1. `Rule First`：关键结论（金额、时效、高风险判断）优先由规则引擎生成。
2. `Evidence First`：结论必须绑定“数据来源+规则依据+法条检索依据”。
3. `Human in the Loop`：敏感场景（工伤等级、时效边缘、高额争议）强制“需复核”。
4. `Fail Safe`：外部服务失败时，系统降级为本地规则输出，不中断主链路。
5. `Security by Design`：最小采集、分级保护、可追溯日志、可删除机制。

---

## 3. 六层总体架构

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ 交互层                                                                   │
│ Web/H5, 小程序(可选), 律师端, 运营后台                                   │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────────────┐
│ 业务层                                                                   │
│ 工作流引擎(按案型), 会话编排, 任务状态机, 繁简分流, 推荐路由              │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────────────┐
│ 规则层                                                                   │
│ 本地化规则包, 计算规则单元, 文书模板映射, 风险规则, 版本管理              │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────────────┐
│ 模型层                                                                   │
│ OpenHarness Agent/Session/Middleware, LLM Provider, MCP Tooling          │
│ 北大法宝 MCP(法规/案例/实务检索), 结构化抽取与生成                        │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────────────┐
│ 数据层                                                                   │
│ 业务库(用户/案件/会话), 规则库, 模板库, 日志审计库, 向量检索(可选)         │
└───────────────┬──────────────────────────────────────────────────────────┘
                │
┌───────────────▼──────────────────────────────────────────────────────────┐
│ 治理层                                                                   │
│ 权限、审计、质检、观测、告警、版本回滚、合规与留痕                        │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. OpenHarness 在本项目中的职责定位

## 4.1 作为“编排内核”

1. `Agent`：定义智能体能力边界（问诊、抽取、文书、检索整合）。
2. `Session`：托管多轮状态、上下文压缩、自动重试、持久化。
3. `Middleware`：组合重试、压缩、持久化、hooks，实现可插拔策略。
4. `mcpServers`：接入北大法宝 MCP 服务，让模型调用法律检索工具。
5. `approve`：对高风险工具调用做执行前审批（例如写入、外发）。

## 4.2 角色划分建议

1. `triage-agent`：主智能体，负责问诊流程、案型识别、任务编排。
2. `retrieval-agent`：专注法规/案例检索及证据化引用（可作为子智能体）。
3. `doc-agent`：专注文书段落组装与润色，严格结构模板输出。
4. `risk-agent`：高风险内容复核建议生成（时效、金额、工伤争议）。

---

## 5. 北大法宝 MCP 接入框架

## 5.1 接入目标

1. 让模型在回答法规依据、案例依据、流程依据时调用 MCP 工具。
2. 每条高价值结论都可追溯到“检索结果 + 规则版本 + 生成时间”。
3. 降低自由生成法条编号错误、虚构案例、过时依据等风险。

## 5.2 接入模式

1. 单服务器模式：仅接北大法宝 MCP，工具名不需要前缀。
2. 多服务器模式：接多个 MCP 服务时，使用 `serverName_toolName` 规避冲突。
3. 生产建议：优先 `http`（OpenHarness 文档推荐生产使用），`sse` 兼容历史链路。

## 5.3 MCP 配置模板（示例）

注意：实际 URL、headers、token 以北大法宝控制台下发配置为准。

```ts
import { Agent } from "@openharness/core";
import { openai } from "@ai-sdk/openai";

export const triageAgent = new Agent({
  name: "triage-agent",
  model: openai("gpt-5.4"),
  systemPrompt: "你是劳动争议分诊助手。关键结论必须引用检索依据或规则依据。",
  tools: {
    // 本地规则工具 + 结构化工具（下文定义）
  },
  mcpServers: {
    pkulaw: {
      type: "http", // 或 "sse"，按控制台配置
      url: process.env.PKULAW_MCP_URL!,
      headers: {
        Authorization: `Bearer ${process.env.PKULAW_MCP_TOKEN!}`,
      },
    },
  },
  maxSteps: 25,
});
```

---

## 6. 六层落地设计（详细）

## 6.1 交互层

### 6.1.1 功能

1. 用户问诊入口：支持口述式输入、分段追问、事实回显确认。
2. 结果展示：双层输出（大白话结论 + 专业依据层）。
3. 文件上传：劳动合同、工资流水截图、聊天记录等材料。
4. 决策入口：继续自助 / 联系律师 / 补充材料 / 复核咨询。

### 6.1.2 页面建议

1. `consultation`: 问诊流程页。
2. `result`: 测算+文书+证据建议页。
3. `triage`: 繁简分流与推荐页。
4. `admin`: 规则、模板、审核、日志看板。

## 6.2 业务层

### 6.2.1 工作流状态机

```text
INIT
 -> FACT_COLLECTION
 -> FACT_CONFIRMATION
 -> CASE_CLASSIFICATION
 -> RULE_CALCULATION
 -> MCP_EVIDENCE_RETRIEVAL (按需触发)
 -> DOCUMENT_ASSEMBLY
 -> TRIAGE_DECISION
 -> FINAL_OUTPUT
```

### 6.2.2 案型驱动编排

1. 未签合同：重点追问合同签署时间、工资支付周期、证据链。
2. 违法解除：重点追问解除理由、通知方式、在职时长、时效边界。
3. 工资拖欠：重点追问拖欠金额、周期、支付记录。
4. 社保争议：重点追问参保状态、欠缴周期、单位名称一致性。
5. 工伤：重点追问事故时间、医疗材料、工伤认定进度。

## 6.3 规则层

### 6.3.1 规则单元结构

```yaml
rule_id: xian.illegal_termination.compensation.v1
region: xian
effective_from: 2026-01-01
inputs:
  - labor_months
  - monthly_salary
  - termination_type
logic:
  formula: "monthly_salary * N * 2"
  constraints:
    - "N <= 12"
outputs:
  - amount
  - explanation
risk_level: high
requires_review: false
```

### 6.3.2 规则版本治理

1. 规则版本号：`region.domain.rule.major.minor`。
2. 生效时间：支持未来生效与灰度发布。
3. 审核流程：法务审核 -> 技术审核 -> 发布。
4. 可回滚：保留最近 N 个稳定版本。

## 6.4 模型层

### 6.4.1 OpenHarness Session 策略

```ts
import { Session } from "@openharness/core";

const session = new Session({
  agent: triageAgent,
  contextWindow: 200_000,
  retry: {
    maxRetries: 4,
    initialDelayMs: 1000,
    maxDelayMs: 20000,
  },
});
```

### 6.4.2 Middleware 策略

1. `withTurnTracking`：记录每轮编号、耗时、令牌。
2. `withCompaction`：上下文过长时压缩历史，保留关键字段。
3. `withRetry`：429/5xx 自动退避重试。
4. `withPersistence`：每轮写会话存储，支持断点续聊。
5. `withHooks`：埋点、风控拦截、质量标记。

### 6.4.3 结构化输出约束

1. 强制 JSON Schema 输出，避免段落式自由发挥。
2. 关键字段不全时返回 `missing_fields`，触发继续追问。
3. 高风险结论必须包含 `risk_label` 与 `review_required`。

## 6.5 数据层

### 6.5.1 核心实体

1. `user_profile`：用户基础信息（脱敏存储）。
2. `case_intake`：案件主表（案型、地区、阶段、风险等级）。
3. `fact_snapshot`：事实结构化快照（版本化）。
4. `calc_result`：测算结果与规则版本映射。
5. `mcp_reference`：检索引用（工具名、查询词、摘要、来源链接）。
6. `document_output`：文书草稿与模板版本。
7. `triage_result`：繁简分流与推荐记录。
8. `audit_log`：操作与模型调用审计日志。

### 6.5.2 关系型表设计（示例）

```sql
CREATE TABLE case_intake (
  case_id            BIGSERIAL PRIMARY KEY,
  user_id            BIGINT NOT NULL,
  region_code        VARCHAR(32) NOT NULL,
  case_type          VARCHAR(64) NOT NULL,
  stage              VARCHAR(32) NOT NULL,
  complexity_level   VARCHAR(16),
  risk_level         VARCHAR(16),
  created_at         TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at         TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE mcp_reference (
  ref_id             BIGSERIAL PRIMARY KEY,
  case_id            BIGINT NOT NULL,
  provider           VARCHAR(64) NOT NULL,      -- pkulaw
  tool_name          VARCHAR(128) NOT NULL,
  query_text         TEXT NOT NULL,
  citation_text      TEXT,
  source_url         TEXT,
  retrieved_at       TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 6.6 治理层

### 6.6.1 质量治理

1. 结论分级：`system_estimate` / `need_lawyer_review` / `blocked`.
2. 自动质检：金额范围、法条格式、时间逻辑一致性。
3. 人工抽检：按日采样高风险会话，形成质检报告。

### 6.6.2 安全合规治理

1. 敏感字段加密：身份证、手机号、银行卡、病历信息。
2. 最小权限：运营只看脱敏摘要，律师仅看授权案件。
3. 全链路留痕：登录、查看、导出、转介全部记录。
4. 生命周期：默认保存期 + 用户删除请求流程。

---

## 7. 十大能力逐项实现方案

## 7.1 六层架构搭建

实施步骤：

1. 先建立空骨架工程，定义层间接口契约。
2. 业务层仅依赖规则层和模型层抽象接口，不耦合具体厂商。
3. 数据层先建主表与审计表，再接入对象存储与日志系统。
4. 治理层从 Day 1 接入日志与权限，不后置。

验收标准：

1. 单层替换不影响其他层。
2. 核心链路可观测（traceId 贯通）。

## 7.2 劳动争议工作流引擎

实施步骤：

1. 建立案型路由器：根据关键词+字段命中进行案型识别。
2. 定义每个案型的 `required_fields` 与 `question_plan`。
3. 引入“确认节点”：事实抽取后必须让用户确认时间线。
4. 输出状态机快照，支持中断恢复。

验收标准：

1. 任一案型可完成“提问-确认-输出”闭环。
2. 缺失字段时能自动继续追问，而非直接给结论。

## 7.3 本地化赔偿计算引擎

实施步骤：

1. 把计算拆成规则单元（适用条件、公式、边界、解释模板）。
2. 将“参数”与“逻辑”分离，地区参数独立配置。
3. 输出结果同时附带规则版本和计算过程文本。
4. 对高风险项标记 `requires_review=true`。

验收标准：

1. 可根据地区切换规则包。
2. 计算结果可解释、可复算、可追溯。

## 7.4 模型与检索增强（MCP）

实施步骤：

1. 在 Agent 中挂载 `mcpServers.pkulaw`。
2. 设置检索触发条件（问法条、问案例、问流程依据）。
3. 检索结果结构化存储到 `mcp_reference`。
4. 输出时展示“引用依据”和“检索时间”。

验收标准：

1. 法律依据类问答可返回可核查引用。
2. MCP 不可用时自动降级，主流程不中断。

## 7.5 事实抽取与案情结构化

实施步骤：

1. 定义统一事实 Schema（身份、薪酬、合同社保、事件、证据、风险）。
2. 首轮抽取后输出 `confidence` 与 `missing_fields`。
3. 对关键字段做规则校验（日期合法、金额非负、时间先后关系）。
4. 生成案情卡 + 事实时间轴。

验收标准：

1. 字段完整率、字段准确率达到验收阈值。
2. 时间轴可用于后续文书自动填充。

## 7.6 文书模板系统

实施步骤：

1. 模板分三层：结构模板、段落模板、语言润色模板。
2. 用字段填充生成初稿，禁止完全自由生成全文。
3. 保留“待人工填写”占位符，避免伪造未知信息。
4. 导出 Markdown/PDF/Word（可选）。

验收标准：

1. 文书字段覆盖率可量化。
2. 律师可在后台对模板版本进行管理和回滚。

## 7.7 繁简分流与推荐机制

实施步骤：

1. 定义复杂度评分维度：金额、证据完整度、法律关系复杂度、时效风险。
2. 计算综合评分后给出策略：自助/建议律师/紧急转介。
3. 推荐逻辑加权：案型匹配、地域匹配、服务能力标签。
4. 记录推荐日志，支持后续转化分析。

验收标准：

1. 分流结果可解释（可输出评分明细）。
2. 推荐结果可回溯至具体规则。

## 7.8 质量控制体系

实施步骤：

1. 输出后置校验器：法条引用格式、金额合理性、术语一致性。
2. 高风险标记强制追加“需复核提示”。
3. 失败分类：模型失败/检索失败/规则失败。
4. 建立日常质量面板（通过率、复核率、驳回率）。

验收标准：

1. 幻觉率与错误引用率可持续下降。
2. 任一结论都可追踪来源。

## 7.9 数据安全与合规机制

实施步骤：

1. 数据分级：公开、内部、敏感、严格敏感。
2. 敏感字段加密与脱敏展示并行。
3. 细粒度授权：用户授权后律师可见。
4. 提供“导出/删除/撤回授权”能力。

验收标准：

1. 能证明“谁在何时访问了哪些数据”。
2. 用户可发起数据删除并有闭环记录。

## 7.10 后台治理能力

实施步骤：

1. 规则管理：发布、回滚、灰度、审批流。
2. 模板管理：版本比对、审核、归档。
3. 风险看板：高风险会话、待复核会话、错误热点。
4. 运营看板：转化漏斗、分流分布、推荐命中率。

验收标准：

1. 运营/法务/技术角色权限隔离明确。
2. 关键治理动作全部留痕可审计。

---

## 8. OpenHarness 工程目录建议（独立项目）

```text
smart-triage/
  apps/
    web/                          # 用户端
    admin/                        # 后台端
    api/                          # BFF/业务API
  packages/
    agent-core/                   # OpenHarness Agent/Session/Middleware
    workflow-engine/              # 状态机与编排
    rule-engine/                  # 规则加载/执行/版本管理
    legal-retrieval/              # 北大法宝 MCP 封装
    document-engine/              # 文书模板生成
    risk-engine/                  # 分流与风险评分
    shared-schema/                # zod/json schema 与类型定义
  infra/
    docker/
    k8s/
  docs/
    architecture.md
    api-contracts.md
    compliance.md
```

---

## 9. 核心接口设计（示例）

## 9.1 会话问诊

`POST /api/v1/cases/{caseId}/chat`

请求体：

```json
{
  "message": "公司口头辞退我，没给书面通知",
  "attachments": [],
  "client_ts": 1760000000000
}
```

响应体（流式事件）：

```json
{
  "event": "fact_extracted",
  "payload": {
    "case_type": "illegal_termination",
    "missing_fields": ["entry_date", "monthly_salary"],
    "risk_label": "medium"
  }
}
```

## 9.2 生成输出包

`POST /api/v1/cases/{caseId}/generate`

请求体：

```json
{
  "output_types": [
    "calculation_statement",
    "arbitration_draft",
    "evidence_directory",
    "action_checklist"
  ]
}
```

## 9.3 繁简分流

`POST /api/v1/cases/{caseId}/triage`

响应体：

```json
{
  "complexity": "moderate",
  "score_breakdown": {
    "amount": 20,
    "evidence": 15,
    "legal_complexity": 30,
    "deadline_risk": 10
  },
  "recommendation": "self_help_plus_lawyer_optional"
}
```

---

## 10. 端到端执行链路（生产建议）

1. 用户输入叙述 -> OpenHarness Session 接收消息。
2. 主智能体调用 `extract_facts` 自定义工具，产出结构化字段。
3. 字段不全则继续追问；字段齐全后进入规则计算。
4. 若用户需要依据，触发北大法宝 MCP 检索工具。
5. 合并规则结果与检索依据，生成双层输出。
6. 进入繁简分流，给出自助或律师建议。
7. 全程事件写入审计日志与质量指标流。

---

## 11. 分阶段实施计划（12 周）

## 11.1 Phase 0（第 1-2 周）：基础底座

1. 初始化工程、CI/CD、环境分层（dev/staging/prod）。
2. 接入 OpenHarness Agent + Session 最小可运行版本。
3. 接通北大法宝 MCP 基础连通性（list tools / call tools）。
4. 建立基础日志、trace、错误码规范。

交付物：

1. 可跑通的问答 + MCP 检索 demo。
2. 基础监控看板。

## 11.2 Phase 1（第 3-5 周）：核心链路 MVP

1. 工作流引擎（2-3 个高频案型）。
2. 本地化赔偿计算引擎 v1（规则单元化）。
3. 事实抽取 schema 与缺失字段追问。
4. 文书模板系统 v1（仲裁草稿+证据目录+行动清单）。

交付物：

1. “问诊-计算-文书-分流”闭环可演示版本。

## 11.3 Phase 2（第 6-9 周）：质量与治理强化

1. 质量校验器与高风险策略。
2. 后台规则/模板审核与版本回滚。
3. 分流推荐引擎与律师标签体系。
4. 合规机制（脱敏、授权、删除流程）。

交付物：

1. 内测版（支持真实小流量灰度）。

## 11.4 Phase 3（第 10-12 周）：试点与优化

1. A/B 测试（提问策略、结果展示样式）。
2. 性能优化（缓存、限流、重试策略调优）。
3. 转化漏斗分析与推荐模型迭代。
4. 上线准备与应急预案演练。

交付物：

1. 生产候选版本（Release Candidate）。

---

## 12. 测试与验收体系

## 12.1 测试类型

1. 单元测试：规则计算、字段抽取、模板填充。
2. 集成测试：OpenHarness + MCP + 规则引擎端到端。
3. 回归测试：规则更新前后结果一致性。
4. 安全测试：权限越权、敏感信息泄露、注入风险。
5. 人工评审：律师对文书可用度、依据可信度打分。

## 12.2 核心验收指标（建议）

1. 事实字段完整率 >= 90%（核心字段集合）。
2. 关键金额计算一致率 >= 95%（与人工规则核对集）。
3. 检索引用命中率 >= 90%（法规/案例依据问法）。
4. 高风险错误出街率 < 1%（上线阈值可更严）。
5. P95 首次结果返回时延 <= 6s（无附件场景）。

---

## 13. 部署与运维建议

1. API 与 Agent 服务分离部署，便于横向扩展。
2. MCP 调用做独立网关层，统一限流、鉴权、审计。
3. 缓存策略：规则包缓存、模板缓存、热点检索缓存。
4. 观测体系：日志 + 指标 + Trace 三件套。
5. 应急降级：
   - MCP 异常 -> 仅输出规则结果并提示依据延迟。
   - 模型异常 -> 回退至规则问答模板。
   - 规则异常 -> 拦截高风险输出并转人工。

---

## 14. 环境变量清单（建议）

```bash
# LLM
OPENAI_API_KEY=
OPENAI_MODEL=gpt-5.4

# OpenHarness
AGENT_MAX_STEPS=25
SESSION_CONTEXT_WINDOW=200000

# PKULAW MCP
PKULAW_MCP_URL=
PKULAW_MCP_TOKEN=
PKULAW_MCP_TRANSPORT=http

# Security
APP_ENCRYPTION_KEY=
JWT_SECRET=
RBAC_POLICY_VERSION=v1

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=
LOG_LEVEL=info
```

---

## 15. 风险清单与应对

1. 规则更新滞后风险：建立每周规则审查节奏 + 紧急发布通道。
2. 检索依赖风险：缓存热门依据 + 多级降级策略。
3. 模型波动风险：固定模型版本 + 回归样本集日跑。
4. 合规风险：默认脱敏展示 + 严格授权 + 可删可查。
5. 误导性输出风险：高风险项统一 `需复核` 标识并引导律师介入。

---

## 16. 上线前“必过清单”

1. 关键案型回归样本全部通过。
2. 高风险场景全部触发复核提示。
3. 所有输出可追溯到规则版本或检索引用。
4. 删除授权、删除数据、导出记录流程可用。
5. 压测达到目标并具备应急降级演练记录。

---

## 17. 参考规范（官方）

1. OpenHarness 文档（Introduction）：https://docs.open-harness.dev/
2. OpenHarness MCP Servers：https://docs.open-harness.dev/advanced/mcp-servers
3. OpenHarness Sessions：https://docs.open-harness.dev/core/sessions
4. OpenHarness Middleware：https://docs.open-harness.dev/core/middleware
5. 北大法宝 MCP 平台首页：https://mcp.pkulaw.com/

> 说明：北大法宝的具体 MCP 工具列表、接入 URL、Header 及权限项以控制台实际下发配置为准。

