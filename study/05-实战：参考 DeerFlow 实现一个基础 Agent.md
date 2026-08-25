# 05｜实战：参考 DeerFlow 实现一个基础 Agent

> 目标：让你从“看懂项目”走到“能自己做一个基础 Agent”。

---

## 1. 先定范围：什么叫“基础 Agent”

本章定义的基础 Agent 能力：

- 接收用户目标；
- 使用 LLM 决策；
- 调用至少 2 个工具（例如 `read_file` + `bash`）；
- 有最小状态（messages + artifacts）；
- 能流式返回结果。

不要求你一开始实现：

- 复杂技能系统；
- 多租户鉴权；
- 完整子智能体编排。

---

## 2. 先画你的“最小架构”

建议直接借鉴 DeerFlow 的四段式：

1. **入口层**：接请求（类似 `thread_runs.py`）；
2. **服务层**：`start_run` 风格的统一调度；
3. **Agent 组装层**：模型 + 工具 + 中间件；
4. **执行层**：后台运行 + 流式事件。

为什么这样做：

- 即使做最小版本，也要先把层次打对；
- 后续加记忆、技能、子智能体会非常自然。

---

## 3. 实施步骤（按 DeerFlow 思路）

### 第 1 步：定义最小状态

参考：`backend/packages/harness/deerflow/agents/thread_state.py`

你可以先定义：

- `messages`
- `artifacts`
- `run_meta`

并为 `artifacts` 写 reducer（去重 + 保序）。

### 第 2 步：实现工具装配函数

参考：`backend/packages/harness/deerflow/tools/tools.py`

先做一个最简版 `get_available_tools()`：

- 固定返回 2~3 个工具；
- 做一次同名去重；
- 做一次安全 gate（比如禁用危险命令）。

### 第 3 步：实现 Agent 组装函数

参考：

- `backend/packages/harness/deerflow/agents/factory.py`
- `backend/packages/harness/deerflow/agents/lead_agent/agent.py`

最小实现：

- 选模型；
- 注入工具；
- 注入 1~2 个中间件（错误处理、日志）；
- 调 `create_agent(...)` 返回图。

### 第 4 步：实现 run 生命周期

参考：`backend/app/gateway/services.py`

你的 `start_run` 至少要包含：

- 创建 run 记录；
- 构造运行参数；
- 后台 task 执行 `run_agent`；
- 支持 cancel 或 wait 二选一。

### 第 5 步：实现流式通道

参考：

- `backend/packages/harness/deerflow/runtime/runs/worker.py`
- `backend/app/gateway/services.py`（`sse_consumer`）

最小实现要点：

- 执行线程只负责产出事件；
- SSE 层只负责转发事件；
- 两者通过中间层（队列/bridge）连接。

---

## 4. 一个可抄的“最小开发清单”

你可以按以下清单落地（每完成一项都能跑通一次）：

1. 定义 `ThreadState` 最小字段；
2. 实现 `get_available_tools`；
3. 实现 `make_lead_agent`；
4. 实现 `start_run` + 后台任务；
5. 实现 stream endpoint；
6. 跑通“输入 -> 工具调用 -> 输出”；
7. 补最小测试（工具去重、状态合并、run 终态）。

---

## 5. 你可以直接复用的“设计决策模板”

当你在自己的项目里犹豫怎么做时，可以按这个模板问：

1. **这个能力是业务逻辑还是横切逻辑？**
   - 横切逻辑优先中间件化；
2. **这个状态会不会被并发更新？**
   - 会，就必须 reducer 化；
3. **这个能力是否通过工具暴露给模型？**
   - 是，就要有可见性和执行双重策略；
4. **这个执行是否是长任务？**
   - 是，就要 run 生命周期 + 流式通道解耦。

---

## 6. 进阶路线（做完基础版后）

你可以按下面顺序增强：

1. 加入 durable context（历史摘要/委派账本）；
2. 加入子智能体委派（先串行，后并行）；
3. 加入技能激活与策略过滤；
4. 加入沙箱隔离与 host gate；
5. 加入 run 事件审计与可视化。

对应参考文档：

- `study/DeerFlow-中间件链设计-深度讲解.md`
- `study/DeerFlow-子智能体系统设计-深度讲解.md`
- `study/DeerFlow-沙箱系统设计-深度讲解.md`

---

## 7. 验收标准（你是否真的学会了）

你可以用这 5 个问题自测：

1. 能否画出自己 Agent 的运行时分层图？
2. 能否解释每个中间件存在的必要性？
3. 能否说明状态字段的 reducer 规则？
4. 能否描述一次 run 从创建到终态的完整生命周期？
5. 能否在不看 DeerFlow 源码时实现一个最小可运行版本？

5 个都能答出来，你已经具备独立实现基础 Agent 的能力。

