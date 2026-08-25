# 12｜Agent 编排模块详解 + 落地方案

> 覆盖模块：Lead Agent 工厂、中间件链、ThreadState、Prompt/Skill 注入机制。

---

## 1. 模块清单与入口

1. Agent 工厂：`backend/packages/harness/deerflow/agents/factory.py`
2. Lead Agent 组装：`backend/packages/harness/deerflow/agents/lead_agent/agent.py`
3. Prompt 模板：`backend/packages/harness/deerflow/agents/lead_agent/prompt.py`
4. 中间件实现：`backend/packages/harness/deerflow/agents/middlewares/`
5. 线程状态：`backend/packages/harness/deerflow/agents/thread_state.py`

---

## 2. Agent 组装真实流程

### 2.1 工厂层

`create_deerflow_agent(...)` 负责接收：

- model
- tools
- middleware
- state_schema
- checkpointer

并返回可执行图。

### 2.2 Lead 业务层

`make_lead_agent(...) -> _make_lead_agent(...)` 决定：

1. 本次 run 使用哪个 model；
2. 是否启用 subagent/plan mode；
3. 允许哪些 tools；
4. 中间件链顺序；
5. 系统提示词注入策略。

---

## 3. 中间件链：执行正确性的关键

### 3.1 你要理解两类钩子

1. **状态钩子**：`before_model` / `after_model`
2. **包装钩子**：`wrap_tool_call`

### 3.2 典型能力如何落在中间件

- 输入清洗：`input_sanitization_middleware.py`
- 动态上下文：`dynamic_context_middleware.py`
- 技能激活：`skill_activation_middleware.py`
- 技能工具策略：`skill_tool_policy_middleware.py`
- durable context：`durable_context_middleware.py`
- 工具错误治理：`tool_error_handling_middleware.py`
- 标题/记忆/图片注入等：对应中间件文件。

### 3.3 顺序为什么关键

- 外层 wrap_tool_call 能否包住内层错误处理决定结果一致性；
- policy 过滤必须早于工具实际暴露；
- summarization 位置影响上下文质量与 token 成本。

---

## 4. ThreadState：状态契约而非随意字典

关键文件：`backend/packages/harness/deerflow/agents/thread_state.py`

核心字段：

- `sandbox`
- `thread_data`
- `artifacts`
- `promoted`
- `delegations`
- `skill_context`
- `summary_text`

核心机制：reducer 合并。

示例价值：

- `merge_promoted` 用 `catalog_hash` 防止工具目录漂移；
- `merge_delegations` 防止 terminal 状态被非 terminal 覆盖；
- `merge_sandbox` 防止同线程 sandbox id 冲突。

---

## 5. 为什么这么设计

1. **工厂分层**：通用拼装与业务策略分开；
2. **中间件链**：横切能力可组合、可插拔、可测试；
3. **状态 reducer**：并发与多步更新下保持确定性。

---

## 6. 完整落地方案（按 DeerFlow 模式复刻）

### 6.1 先实现最小 AgentState

定义字段 + reducer：

- `messages`
- `artifacts`
- `delegations`

### 6.2 再实现双层工厂

1. `create_agent_core(...)`：纯参数拼装；
2. `make_lead_agent(...)`：读取 run config，决策工具与中间件。

### 6.3 再实现中间件体系

优先上 5 个：

1. 输入清洗
2. 工具错误治理
3. 动态上下文
4. summarization
5. token usage

### 6.4 最后接入技能与策略

1. slash 激活技能
2. 工具策略过滤
3. durable context 注入

---

## 7. 验收标准

- 同一请求在不同运行模式下工具集可控；
- 中间件顺序调整后可被测试及时发现问题；
- 状态字段在并发更新时无随机覆盖；
- 技能策略生效时，模型可见 schema 与执行权限一致。

