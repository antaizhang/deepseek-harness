# 代码地图

[English](code-map.md) | 中文

本页是阅读 `packages/` 之前的定位环节：每项关注点由哪个包拥有，以及一个轮次在源码中经过的有序路径。[architecture.md](architecture.md) 描述这些文件所实现的行为，[各子系统页面](subsystems/README.md)定义它们传递的类型，每个包的 README 陈述自身的契约。本页只保存三者之间的映射，因此每一行都链接到其拥有者，而不复述其内容。

## 谁拥有什么

[核心包](../packages/core/README.md)在彼此之间承载一个轮次；其余每项能力都挂在有文档的扩展点上。

| 关注点 | 源码 | 参考 |
|---|---|---|
| 持久事件日志与模型可见 surface | [`core/session`](../packages/core/session) | [session.md](subsystems/session.md) |
| 提示词分节、动态上下文、工具 schema 与变量 | [`core/system-prompt`](../packages/core/system-prompt) | [system-prompt.md](subsystems/system-prompt.md) |
| 工具注册表与受守卫的执行管线 | [`core/tools`](../packages/core/tools) | [tools.md](subsystems/tools.md) |
| `Agent` 句柄、其收件箱与 `agent/*` 词汇 | [`core/agent`](../packages/core/agent) | [core.md](subsystems/core.md) |
| 具体的轮次与步骤驱动器 | [`core/agent-loop`](../packages/core/agent-loop) | [core.md](subsystems/core.md) |
| 每-agent 的作用域化注册 | [`core/scope`](../packages/core/scope) | [scope.md](subsystems/scope.md) |
| 消息词汇与模型适配器 seam | [`llm/llm`](../packages/llm/llm) | [llm-streaming.md](subsystems/llm-streaming.md) |

这条线以下的一切都是[能力 seam](architecture.md#capability-seams) 或其消费方：[packages/README.md](../packages/README.md) 列出包组，[tool-catalog.md](tool-catalog.md) 列出它们贡献的面向模型的工具。

## 一个轮次在源码中的路径

按顺序阅读以下各行，即可跟随一条提示词从投递走到已持久化的轮次。每一行的行为契约在 [architecture.md](architecture.md#turn-flow) 中，[agent-lifecycle.md](agent-lifecycle.md) 绘制了同一条时序。

| # | 发生了什么 | 源码 |
|---|---|---|
| 1 | 输入进入两条收件箱列表之一，唤醒型投递预留驱动器 | `ReactLoopAgent.send` / `wakeDriver`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 2 | 驱动器在 `withInitiator` 下持续开启轮次，直到不再有欠下的工作 | `ReactLoopAgent.kick`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 3 | 在认领任何输入之前追加 `turn/start` | `ReactLoopAgent.turn`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 4 | 认领本步骤的消息批次，由 `agent/pre-step` 接受或拒绝 | `ReactLoopAgent.preStep`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 5 | 分节、上下文、工具与变量沿作用域链合并 | `SystemPrompt.assemble`，[`system-prompt/src/index.ts`](../packages/core/system-prompt/src/index.ts) |
| 6 | 仅当文本发生变化时才提出动态上下文快照 | `RuntimeContextProjection.project`，[`agent-loop/src/runtime-context.ts`](../packages/core/agent-loop/src/runtime-context.ts) |
| 7 | 进入的消息追加为 `user/message`，`step/start` 开启步骤 | `ReactLoopAgent.turn`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 8 | 解析请求头，在其变化时记录，然后冻结 | `ReactLoopAgent.buildRequest`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 9 | 从当前 surface 推导模型历史 | `Session.deriveMessages`，[`session/src/index.ts`](../packages/core/session/src/index.ts) |
| 10 | 每个分片追加为 `assistant/chunk` 并送入块组装器 | `ReactLoopAgent.step`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 11 | 失败的请求在其步骤关闭前经过 `agent/request-error` | `ReactLoopAgent.step`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 12 | 工具调用按并发模式分组，并按模型顺序提交 | `executeToolCalls`，[`agent-loop/src/tool-calls.ts`](../packages/core/agent-loop/src/tool-calls.ts) |
| 13 | 每次调用依次经过前置执行、守卫、环绕派发与后置执行 | `ToolRuntime.execute`，[`tools/src/index.ts`](../packages/core/tools/src/index.ts) 与 [tool-execution-pipeline.md](tool-execution-pipeline.md) |
| 14 | 结果上下文落入 next-step 列表，随后运行 `agent/turn-stopping` | `ReactLoopAgent.turn`，[`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 15 | 追加 `turn/end`，耐久性由检查点策略拥有 | [`session-checkpoint-policy`](../packages/session/session-checkpoint-policy) |

## 上下文如何构建与缩减

任何到达模型请求的内容都可从日志重建，因此下列每个阶段读写的都是会话事件，而不是私有缓冲区。

| 阶段 | 源码 | 参考 |
|---|---|---|
| 哪些事件到达模型，以及一段区间如何被遮蔽 | [`session/src/surface.ts`](../packages/core/session/src/surface.ts) | [session.md](subsystems/session.md) |
| 工作区指令链及其后续变更 | [`context/agent-instructions`](../packages/context/agent-instructions) | [其 README](../packages/context/agent-instructions/README.md) |
| 目录中的技能摘要，技能正文按需加载 | [`skill/tool-skill`](../packages/skill/tool-skill) | [skills.md](subsystems/skills.md) |
| 请求压力与当前 surface 的计价 | [`llm/token-meter`](../packages/llm/token-meter) | [token-meter.md](subsystems/token-meter.md) |
| 任何摘要之前的无模型工具结果剪枝 | [`compaction/compaction-tool-result-pruner`](../packages/compaction/compaction-tool-result-pruner) | [compaction.md](subsystems/compaction.md) |
| 区间选择、摘要生成与 surface 替换 | [`compaction/compaction-basic`](../packages/compaction/compaction-basic) | [compaction.md](subsystems/compaction.md) |
| 超大工具文本存放在面向模型的定位符之后 | [`spill/spill`](../packages/spill/spill) | [spill.md](subsystems/spill.md) |
| 持久存储、恢复与崩溃后重建 | [`session/session-persistence`](../packages/session/session-persistence) | [persistence.md](subsystems/persistence.md) |

## 接下来去哪里

新行为挂在扩展点上，而不是挂在这些文件里：[architecture.md](architecture.md#where-new-behavior-goes) 将每个目标映射到其机制，[扩展 cookbook](cookbook/extension-cookbook.md) 索引了[包](cookbook/adding-a-package.md)与[工具](cookbook/adding-a-tool.md)的分步指南。
