# 01｜零基础入门：Agent 概念与 DeerFlow 映射

## 1. 先回答：Agent 到底是什么？

对零基础同学，一个实用定义是：

> **Agent = 能围绕目标“持续行动”的程序，而不是一次性回复的聊天函数。**

它通常具备 6 个核心部分：

1. **目标（Goal）**：用户想让它完成什么；
2. **大脑（Model）**：LLM 负责推理和决策；
3. **手脚（Tools）**：调用外部能力执行动作；
4. **记忆（State/Memory）**：记住上下文和历史；
5. **执行循环（Loop）**：观察 → 决策 → 执行 → 再观察；
6. **运行时（Runtime）**：把上面 5 点组织起来的系统底座。

---

## 2. 在 DeerFlow 里这些概念对应什么代码？

| Agent 概念 | DeerFlow 对应实现 | 为什么这样分层 |
|---|---|---|
| 目标输入 | `frontend/src/core/threads/hooks.ts` + `backend/app/gateway/routers/thread_runs.py` | 前后端解耦，前端只管交互，后端统一调度 |
| 模型选择 | `backend/packages/harness/deerflow/models/factory.py` | 模型是可替换资源，不应和业务逻辑耦合 |
| 工具系统 | `backend/packages/harness/deerflow/tools/tools.py` | 工具是能力插件，需要统一注册、筛选、去重 |
| 执行引擎 | `backend/packages/harness/deerflow/runtime/runs/worker.py` | 执行循环在统一 worker 中管理，便于并发/取消/流式输出 |
| Agent 组装 | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | 将提示词、工具、中间件拼成“可运行 agent” |
| 状态记忆 | `backend/packages/harness/deerflow/agents/thread_state.py` | 明确定义状态结构，避免“隐式魔法状态” |

---

## 3. DeerFlow 不是“聊天机器人”，区别在哪？

常见聊天机器人流程：

`输入 -> LLM -> 输出`

DeerFlow 主流程：

`输入 -> Lead Agent -> 工具调用/子智能体/状态更新 -> 多轮循环 -> 输出`

也就是说 DeerFlow 的关键价值不只是“会回答”，而是：

- 能操作文件与命令（通过沙箱工具）；
- 能根据运行状态继续决策；
- 能把复杂任务拆给子智能体；
- 能在长链路中持续保持可观测状态（run、event、checkpoint）。

---

## 4. 你需要先建立的三个思维模型

### 模型 A：Agent 是“状态机”，不是“函数调用”

- 函数调用更像“一次请求一次返回”；
- Agent 更像“不断读写状态的循环系统”。

在 DeerFlow 中，这个“状态机的状态容器”就是 `ThreadState`（见 `backend/packages/harness/deerflow/agents/thread_state.py`）。

### 模型 B：工具不是附属品，而是 Agent 的执行面

没有工具，LLM 只能生成文本；
有工具，Agent 才能把任务“落地执行”。

`backend/packages/harness/deerflow/tools/tools.py` 里可以看到工具装配逻辑：

- 按配置加载工具；
- 处理 host bash 安全策略；
- 按模型能力决定是否注入视觉工具；
- 加入内置工具（如 `task`）与 MCP 工具。

### 模型 C：运行时比“写 prompt”更重要

很多初学者会把 Agent 开发理解成“写一个大 prompt”。

DeerFlow 的代码告诉我们：

- prompt 只是一个输入；
- 真正决定系统可用性的，是运行时能力（并发、超时、流式、取消、状态一致性、权限边界）。

这些能力主要落在：

- `backend/app/gateway/services.py`
- `backend/packages/harness/deerflow/runtime/runs/worker.py`
- `backend/packages/harness/deerflow/runtime/runs/manager.py`

---

## 5. 为什么要有“Lead Agent + Subagent”

在 DeerFlow 中：

- Lead Agent 负责全局目标和收敛；
- Subagent 负责局部任务执行（通过 `task` 工具委派）。

对应代码：`backend/packages/harness/deerflow/tools/builtins/task_tool.py`

这样设计的原因是：

1. **控制复杂度**：主上下文不被细节淹没；
2. **并行效率**：多个子任务可以并发处理；
3. **可恢复**：子任务失败不一定拖垮主任务；
4. **可观测**：子任务状态可单独跟踪。

---

## 6. 本章你至少应掌握的结论

学完本章，请你能复述：

- Agent 是目标驱动、状态驱动、工具驱动的执行系统；
- DeerFlow 的核心不在“单次回答”，而在“完整运行时链路”；
- 关键源码要从“入口（run）→ 组装（agent）→ 执行（worker）→ 状态（thread_state）”去看。

下一章我们会按真实代码，把整条链路从前端输入拆到最终输出。

