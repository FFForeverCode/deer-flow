# DeerFlow Agent 学习教程（零基础路径）

> 目标：把 `study/` 从“源码说明集合”升级为“可自学、可落地、可复现”的 Agent 系统化教程。

## 架构与流程图总览

阅读 [16｜DeerFlow 整体架构与业务流程图](./16-DeerFlow整体架构与业务流程图.md)，可查看基于当前代码核对的系统架构、部署拓扑、Web 对话时序、Agent 编排、子智能体协作、文件交付、多渠道接入、定时调度、MCP 长任务及状态持久化图。

---

## 你将学会什么

完整读完本目录后，你应该可以：

- 说清楚 Agent 的核心组成（模型、工具、状态、执行循环、记忆、调度）；
- 理解 DeerFlow 的整体架构和模块边界（Frontend / Gateway / Harness / Sandbox）；
- 跟着真实代码走完“用户输入 → 最终输出”的执行链路；
- 看懂主智能体（Lead Agent）是如何被构建、配置并运行的；
- 理解关键设计“为什么这么做”（而不只是知道“怎么做”）；
- 参考当前项目独立实现一个基础 Agent。

---

## 推荐学习顺序（从零开始）

1. `study/01-零基础入门：Agent 概念与 DeerFlow 映射.md`
2. `study/02-主链路拆解：从用户输入到最终输出.md`
3. `study/03-核心源码导读：Lead Agent 如何构建与运行.md`
4. `study/04-为什么这样设计：DeerFlow 的关键设计模式.md`
5. `study/05-实战：参考 DeerFlow 实现一个基础 Agent.md`

---

## 如何边读边看源码（强烈建议）

建议一边看教程一边打开以下关键文件：

- 前端发送消息入口：`frontend/src/core/threads/hooks.ts`
- 前端 API 客户端：`frontend/src/core/api/api-client.ts`
- Run 路由：`backend/app/gateway/routers/thread_runs.py`
- Run 生命周期服务：`backend/app/gateway/services.py`
- 后台执行 worker：`backend/packages/harness/deerflow/runtime/runs/worker.py`
- Lead Agent 构建：`backend/packages/harness/deerflow/agents/lead_agent/agent.py`
- 通用 Agent 工厂：`backend/packages/harness/deerflow/agents/factory.py`
- 工具装配：`backend/packages/harness/deerflow/tools/tools.py`
- 子智能体工具：`backend/packages/harness/deerflow/tools/builtins/task_tool.py`
- 线程状态：`backend/packages/harness/deerflow/agents/thread_state.py`

---

## 与已有深度文档的关系

本系列是“学习主线”。

你在学完主线后，可以按专题继续深入：

- 中间件专题：`study/DeerFlow-中间件链设计-深度讲解.md`
- 沙箱专题：`study/DeerFlow-沙箱系统设计-深度讲解.md`
- 工具专题：`study/DeerFlow-工具系统设计-深度讲解.md`
- 子智能体专题：`study/DeerFlow-子智能体系统设计-深度讲解.md`
- AOP 对照：`study/DeerFlow-中间件钩子实现原理与AOP对比.md`

可以把它理解为：

- 本系列解决“从 0 到 1 学会做 Agent”；
- 旧专题解决“对某一子系统做深潜与面试级表达”。

---

## 全模块完整版（核心模块 + 落地实现方案）

如果你希望按“系统设计文档”方式一次看全核心模块，请阅读：

1. `study/10-核心模块全景总览与调用链.md`
2. `study/11-网关与运行时模块详解+落地方案.md`
3. `study/12-Agent编排模块详解+落地方案.md`
4. `study/13-能力层模块详解（工具-沙箱-技能-MCP-子智能体）+落地方案.md`
5. `study/14-基础设施模块详解（配置-模型-持久化-记忆-安全-追踪）+落地方案.md`
6. `study/15-前端与多渠道接入模块详解+落地方案.md`
