# 03｜核心源码导读：Lead Agent 如何构建与运行

> 本章重点：不是逐行翻译代码，而是教你如何“带着问题读源码”。

---

## 1. 建议先带着 5 个问题读

1. Lead Agent 的“装配入口”在哪？
2. 本次 run 的模型、工具、中间件是如何决定的？
3. 中间件顺序是谁控制的？
4. ThreadState 为什么是 AgentState 的扩展？
5. create_agent 后，运行时是谁驱动它执行？

---

## 2. 入口 1：`create_deerflow_agent`（通用工厂）

关键文件：`backend/packages/harness/deerflow/agents/factory.py`

这个函数的定位是：

- 把 `model + tools + middleware + state_schema` 组合成可运行 agent；
- 它位于“原始 `langchain.agents.create_agent`”和“DeerFlow 业务装配层”之间。

你可以把它理解为：

> **“通用拼装车间”**，上层（Lead Agent）决定装什么，下层（LangChain）负责怎么跑。

为什么这样设计：

- 把通用能力（工厂拼装）和业务策略（Lead Agent 定制）解耦；
- 便于未来复用工厂构建其他 agent 形态。

---

## 3. 入口 2：`make_lead_agent`（业务装配入口）

关键文件：`backend/packages/harness/deerflow/agents/lead_agent/agent.py`

`make_lead_agent(config)` 最终会进入 `_make_lead_agent(...)`，核心做了四类事：

1. **读取运行配置**：model、thinking、plan_mode、subagent 开关；
2. **能力装配**：`get_available_tools(...)` + deferred MCP + skill policy；
3. **行为装配**：`build_middlewares(...)` 生成完整中间件链；
4. **提示词装配**：`apply_prompt_template(...)` 生成系统提示。

最后调用 `create_agent(...)` 输出图对象。

为什么这是关键：

- 这一步决定了“本次 run 的能力边界”；
- 也是定位线上行为差异（某次 run 为什么没某个工具）的第一检查点。

---

## 4. 工具装配：`get_available_tools` 的 4 层来源

关键文件：`backend/packages/harness/deerflow/tools/tools.py`

工具来源按优先级可理解为：

1. 配置工具（`config.tools`）；
2. 内置工具（present、clarification、task 等）；
3. MCP 工具（缓存加载并打标签）；
4. 扩展工具（如 ACP invoke）。

并且会做：

- host bash gate 控制；
- 按模型能力注入视觉工具；
- 同名去重（issue #1803 根因治理）。

为什么这套机制重要：

- 避免“工具看起来可见但实际上不可调用”；
- 降低模型选择错误工具的概率；
- 把安全边界尽量前移到装配期。

---

## 5. 中间件链：Lead Agent 的“神经系统”

关键文件：

- `backend/packages/harness/deerflow/agents/lead_agent/agent.py`（`build_middlewares`）
- `backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py`（runtime middlewares 构建）

你需要重点理解两件事：

1. DeerFlow 的许多横切能力（输入清洗、上下文注入、工具错误处理、记忆更新、标题生成）都走中间件；
2. 顺序很关键，尤其是 wrap_tool_call 的“洋葱模型”。

为什么必须中间件化：

- 不把安全、可观测、容错散落到业务代码中；
- 可以用组合方式扩展，不必改核心执行引擎。

---

## 6. 状态结构：`ThreadState` 不是普通 DTO

关键文件：`backend/packages/harness/deerflow/agents/thread_state.py`

`ThreadState` 继承 `AgentState`，并引入大量 reducer：

- `merge_sandbox`
- `merge_artifacts`
- `merge_promoted`
- `merge_delegations`
- `merge_skill_context`

为什么是 reducer 而不是覆盖写入：

- Agent 多步骤并发下会发生“同一 key 多次更新”；
- reducer 让冲突处理可控，保证状态演进可预测。

---

## 7. 执行驱动：真正跑起来的是 `run_agent`

关键文件：`backend/packages/harness/deerflow/runtime/runs/worker.py`

很多人看到 `create_agent` 以为结束了，其实那只是“造好了机器”。

让机器运行的是：

- `backend/app/gateway/services.py` 的 `start_run(...)`
- 它创建后台 task 执行 `run_agent(...)`

`run_agent(...)` 会把执行事件持续写入 `StreamBridge`，最终通过 SSE 推回前端。

为什么这一层单独存在：

- 把“图定义”与“运行生命周期”拆开；
- 更容易支持取消、重连、wait/stream 双协议。

---

## 8. 子智能体：为什么被设计成工具而不是特殊语法

关键文件：`backend/packages/harness/deerflow/tools/builtins/task_tool.py`

`task` 以工具形式存在，意味着：

- 对 LLM 来说，子智能体委派就是一次普通工具调用；
- 对运行时来说，可统一接入鉴权、日志、状态、错误处理。

这是一个非常值得借鉴的设计点：

> 用统一抽象（Tool）承载复杂能力（Subagent），降低系统复杂度。

---

## 9. 读源码时的“最小闭环路径”（推荐）

如果你时间有限，按以下顺序看：

1. `backend/app/gateway/routers/thread_runs.py`
2. `backend/app/gateway/services.py`（看 `start_run`）
3. `backend/packages/harness/deerflow/agents/lead_agent/agent.py`
4. `backend/packages/harness/deerflow/tools/tools.py`
5. `backend/packages/harness/deerflow/runtime/runs/worker.py`
6. `backend/packages/harness/deerflow/agents/thread_state.py`

这 6 个文件足够你建立“从请求到结果”的主心智模型。

---

## 10. 本章复盘

你读完后应能做到：

- 说出 Lead Agent 构建入口、执行入口、工具入口分别在哪；
- 理解“中间件顺序”和“状态 reducer”为什么是系统正确性的关键；
- 明白 DeerFlow 的核心工程能力在运行时，而不只在 prompt。

