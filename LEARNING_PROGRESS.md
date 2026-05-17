# Nanobot / Agent Harness Learning Progress

目标：通过 nanobot 的真实代码路径理解 OpenClaw 类 agent harness 的架构设计，掌握从消息入口、上下文构建、模型调用、工具执行、记忆、安全边界到 WebUI/Gateway 的完整闭环。

## 学习原则

- 先跑通主链路，再看扩展点。
- 每一阶段都用“读代码 -> 画数据流 -> 做小练习 -> 复盘问题”的节奏推进。
- 优先理解稳定接口：message bus、agent loop、runner、provider、tool、channel、session。
- 修改代码前先定位边界：核心逻辑尽量小，能力优先放在 channel、tool、skill、MCP。

## 路线总览

| 阶段 | 主题 | 目标 | 状态 |
| --- | --- | --- | --- |
| 0 | 环境与全局地图 | 能启动、跑测试、知道入口和目录职责 | 未开始 |
| 1 | 消息流与 Agent Loop | 理解 inbound -> context -> runner -> outbound | 进行中 |
| 2 | Agent Runner / Harness 核心 | 理解模型调用、工具循环、注入、截断和错误恢复 | 未开始 |
| 3 | 工具系统与安全边界 | 能新增或审查一个 tool，知道文件、网络、shell guard | 未开始 |
| 4 | Provider / Channel 扩展 | 理解适配层如何把外部模型和聊天平台接入核心 | 未开始 |
| 5 | Memory / Session / Context | 理解历史、压缩、Dream memory、长期任务状态 | 未开始 |
| 6 | WebUI / Gateway / OpenAI API | 理解本地 UI、WebSocket、OpenAI-compatible API 的产品层 | 未开始 |

## 阶段 0：环境与全局地图

阅读：
- `README.md`
- `docs/quick-start.md`
- `docs/configuration.md`
- `nanobot/cli/commands.py`
- `pyproject.toml`

练习：
- 运行一个最小测试：`pytest tests/test_package_version.py -v`
- 跑 lint 快速检查：`ruff check nanobot/agent/runner.py nanobot/agent/loop.py`
- 用自己的话写出“nanobot 是哪些子系统拼起来的”。

掌握标准：
- 能解释 CLI、gateway、agent loop、channel manager、provider factory 的大致关系。

## 阶段 1：消息流与 Agent Loop

阅读：
- `nanobot/bus/queue.py`
- `nanobot/bus/events.py`
- `nanobot/channels/base.py`
- `nanobot/channels/manager.py`
- `nanobot/agent/loop.py`

重点问题：
- `InboundMessage` 和 `OutboundMessage` 分别承载什么？
- `MessageBus` 为什么把 channel 和 agent core 解耦？
- `AgentLoop` 的 `TurnState` 状态机如何组织一次 turn？
- command、context build、run、save、respond 分别在哪个状态发生？

练习：
- 画一张数据流：channel receive -> bus inbound -> `AgentLoop` -> `AgentRunner` -> bus outbound -> channel send。
- 找到一条 WebSocket 或 CLI 消息进入系统的路径。

掌握标准：
- 能从一个用户输入追踪到最终回复发出。

## 阶段 2：Agent Runner / Harness 核心

阅读：
- `nanobot/agent/runner.py`
- `nanobot/providers/base.py`
- `nanobot/agent/hook.py`
- `nanobot/agent/progress_hook.py`

重点问题：
- `AgentRunSpec` 是如何把一次执行所需的上下文、工具、模型参数打包的？
- runner 的循环什么时候继续，什么时候结束？
- provider 返回 tool calls 后，工具结果如何回填到 messages？
- context governance 包括哪些动作：orphan tool result 修复、backfill、microcompact、预算截断？
- injection callback 如何处理用户中途追加消息？

练习：
- 用伪代码重写 `AgentRunner.run()` 的主循环。
- 找出工具并发和串行执行的分支。

掌握标准：
- 能解释 agent harness 的最小闭环：messages + tools + provider + loop + tool results。

## 阶段 3：工具系统与安全边界

阅读：
- `nanobot/agent/tools/base.py`
- `nanobot/agent/tools/registry.py`
- `nanobot/agent/tools/filesystem.py`
- `nanobot/agent/tools/shell.py`
- `nanobot/agent/tools/web.py`
- `nanobot/security/network.py`
- `.agent/security.md`

重点问题：
- tool schema 如何暴露给 LLM？
- `cast_params`、`validate_params`、`execute` 的职责是什么？
- `_resolve_path` 如何限制文件系统访问？
- web fetch 如何防 SSRF？
- shell sandbox 和 workspace restriction 分别保护什么？

练习：
- 选一个现有 tool，写出它的参数 schema、执行路径和失败模式。
- 设计一个简单只读 tool，并说明它应该放在哪里、需要哪些测试。

掌握标准：
- 能安全地新增或 review 一个 tool。

## 阶段 4：Provider / Channel 扩展

阅读：
- `nanobot/providers/factory.py`
- `nanobot/providers/registry.py`
- `nanobot/providers/openai_compat_provider.py`
- `nanobot/providers/anthropic_provider.py`
- `nanobot/channels/base.py`
- 任意一个 channel 实现，如 `nanobot/channels/telegram.py` 或 `nanobot/channels/websocket.py`

重点问题：
- provider 如何把不同模型 API 归一成 `LLMResponse` 和 `ToolCallRequest`？
- provider retry、fallback、streaming 的边界在哪里？
- channel 如何把平台消息转成 `InboundMessage`？
- streaming channel 和普通 channel 的差异是什么？

练习：
- 对比两个 provider 的 tool call 解析逻辑。
- 对比两个 channel 的 send / send_delta。

掌握标准：
- 能判断一个新模型/平台应该接入 provider 还是 channel。

## 阶段 5：Memory / Session / Context

阅读：
- `nanobot/session/manager.py`
- `nanobot/session/goal_state.py`
- `nanobot/agent/context.py`
- `nanobot/agent/memory.py`
- `docs/memory.md`
- `.agent/gotchas.md`

重点问题：
- session history 如何保存和恢复？
- context builder 如何组合 system prompt、history、memory、skills、attachments？
- Dream memory 的两阶段 consolidation 解决什么问题？
- context pollution 为什么是架构风险？

练习：
- 找到一次 turn 保存 history 的位置。
- 解释自动压缩和 Dream memory 的区别。

掌握标准：
- 能说明短期历史、长期记忆、技能上下文的边界。

## 阶段 6：WebUI / Gateway / OpenAI API

阅读：
- `nanobot/api/server.py`
- `docs/openai-api.md`
- `docs/websocket.md`
- `webui/src/lib/nanobot-client.ts`
- `webui/src/hooks/useNanobotStream.ts`
- `webui/src/components/thread/ThreadShell.tsx`

重点问题：
- gateway 如何把浏览器请求接入后端 agent？
- WebSocket multiplex 协议承载哪些事件？
- OpenAI-compatible API 如何复用 agent 能力？
- WebUI 如何展示 tool traces、stream、sessions？

练习：
- 从 WebUI composer 追踪到后端 inbound message。
- 运行 `cd webui && bun run test`。

掌握标准：
- 能解释产品层如何包住同一个 harness。

## OpenClaw 对照视角

学习 nanobot 时，持续把每个模块映射到 OpenClaw/agent harness 的常见抽象：

| 通用抽象 | nanobot 位置 | 需要掌握的判断 |
| --- | --- | --- |
| event bus | `nanobot/bus/` | 是否需要异步解耦、多 channel fan-in/fan-out |
| harness loop | `nanobot/agent/loop.py`, `runner.py` | 哪些逻辑属于核心闭环，哪些应外移 |
| model adapter | `nanobot/providers/` | 如何屏蔽 provider 差异 |
| tool runtime | `nanobot/agent/tools/` | schema、参数验证、执行、安全和结果回填 |
| memory/session | `nanobot/session/`, `nanobot/agent/memory.py` | replay、compact、持久化和污染控制 |
| platform adapter | `nanobot/channels/` | 外部消息协议如何归一化 |
| product shell | `webui/`, `nanobot/api/` | UI/API 如何复用核心 harness |

## 学习日志

| 日期 | 进度 | 关键收获 | 待补问题 |
| --- | --- | --- | --- |
| 2026-05-18 | 建立学习路线 | 初步确定从消息流、runner、tools、providers、memory、WebUI 六条线推进 | 待确认你想先从源码阅读、运行调试，还是架构图开始 |
| 2026-05-18 | 阶段 1 导学：消息流主链路 | `BaseChannel._handle_message()` 将平台消息归一化为 `InboundMessage`；`MessageBus` 用 inbound/outbound 两个 async queue 解耦 channel 和 agent core；`AgentLoop.run()` 负责消费 inbound、按 session 派发，并把同 session 的中途追加消息路由到 pending queue；一次 turn 由 `RESTORE -> COMPACT -> COMMAND -> BUILD -> RUN -> SAVE -> RESPOND` 状态机组织 | 需要通过自测确认是否能独立追踪从用户输入到回复发送的路径 |

## 问题池

- 阶段 1 自测：`InboundMessage.session_key` 的默认构成是什么？为什么它对 session 隔离重要？
- 阶段 1 自测：为什么 `MessageBus` 不直接调用 channel 或 agent，而是使用 queue？
- 阶段 1 自测：同一个 session 在 agent 正在处理时又收到一条消息，`AgentLoop.run()` 如何处理？
- 阶段 1 自测：`BUILD` 和 `RUN` 两个状态的边界是什么？

## 本轮评价

2026-05-18：已完成阶段 1 的第一轮导学，但还没有收到你的自测回答，因此只能评价“已接触核心概念，掌握度待验证”。下一步需要你回答问题池中的 4 个问题，我会据此判断是否进入 `AgentRunner` 主循环，还是先补一遍 `AgentLoop` 状态机。

## 掌握度自评

| 能力项 | 1 生疏 | 2 能定位 | 3 能解释 | 4 能修改 | 5 能设计 |
| --- | --- | --- | --- | --- | --- |
| 消息流 | 待自测 |  |  |  |  |
| Agent runner |  |  |  |  |  |
| Tool 系统 |  |  |  |  |  |
| 安全边界 |  |  |  |  |  |
| Provider 扩展 |  |  |  |  |  |
| Channel 扩展 |  |  |  |  |  |
| Memory/session |  |  |  |  |  |
| WebUI/gateway |  |  |  |  |  |
