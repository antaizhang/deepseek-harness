# Code Map

English | [中文](code-map.zh.md)

This page is the orientation pass before reading `packages/`: which package owns each concern, and the ordered path one turn takes through the source. [architecture.md](architecture.md) describes the behavior these files implement, the [subsystems pages](subsystems/README.md) define the types they move, and each package README states its own contract. This page holds only the mapping between them, so every row links to the owner rather than restating it.

## What owns what

The [core packages](../packages/core/README.md) carry one turn between them; every other capability hangs off a documented extension point.

| Concern | Source | Reference |
|---|---|---|
| Durable event log and the model-visible surface | [`core/session`](../packages/core/session) | [session.md](subsystems/session.md) |
| Prompt sections, dynamic contexts, tool schemas, variables | [`core/system-prompt`](../packages/core/system-prompt) | [system-prompt.md](subsystems/system-prompt.md) |
| Tool registry and the guarded execution pipeline | [`core/tools`](../packages/core/tools) | [tools.md](subsystems/tools.md) |
| The `Agent` handle, its inbox, and the `agent/*` vocabulary | [`core/agent`](../packages/core/agent) | [core.md](subsystems/core.md) |
| The concrete turn and step driver | [`core/agent-loop`](../packages/core/agent-loop) | [core.md](subsystems/core.md) |
| Per-agent scoped registration | [`core/scope`](../packages/core/scope) | [scope.md](subsystems/scope.md) |
| Message vocabulary and the model adapter seam | [`llm/llm`](../packages/llm/llm) | [llm-streaming.md](subsystems/llm-streaming.md) |

Everything below that line is a [capability seam](architecture.md#capability-seams) or a Consumer of one: [packages/README.md](../packages/README.md) lists the groups, and [tool-catalog.md](tool-catalog.md) lists the model-facing tools they contribute.

## One turn through the source

Read these in order to follow one prompt from delivery to a persisted turn. Each row's behavioral contract is in [architecture.md](architecture.md#turn-flow), and [agent-lifecycle.md](agent-lifecycle.md) draws the same sequence.

| # | What happens | Source |
|---|---|---|
| 1 | Input enters one of the two inbox lists, and a waking delivery reserves the driver | `ReactLoopAgent.send` / `wakeDriver`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 2 | The driver opens turns under `withInitiator` until nothing is owed | `ReactLoopAgent.kick`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 3 | `turn/start` is appended before any input is claimed | `ReactLoopAgent.turn`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 4 | The step batch is claimed and `agent/pre-step` accepts or rejects it | `ReactLoopAgent.preStep`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 5 | Sections, contexts, tools, and variables merge across the scope chain | `SystemPrompt.assemble`, [`system-prompt/src/index.ts`](../packages/core/system-prompt/src/index.ts) |
| 6 | A dynamic context snapshot is proposed only when its text changed | `RuntimeContextProjection.project`, [`agent-loop/src/runtime-context.ts`](../packages/core/agent-loop/src/runtime-context.ts) |
| 7 | Entered messages append as `user/message`, and `step/start` opens the step | `ReactLoopAgent.turn`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 8 | The request header is resolved, logged when it changes, and frozen | `ReactLoopAgent.buildRequest`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 9 | Model history is derived from the current surface | `Session.deriveMessages`, [`session/src/index.ts`](../packages/core/session/src/index.ts) |
| 10 | Each chunk appends as `assistant/chunk` and feeds the block assembler | `ReactLoopAgent.step`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 11 | A failed request passes through `agent/request-error` before its step closes | `ReactLoopAgent.step`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 12 | Tool calls are grouped by concurrency mode and committed in model order | `executeToolCalls`, [`agent-loop/src/tool-calls.ts`](../packages/core/agent-loop/src/tool-calls.ts) |
| 13 | Each call runs pre-execute, guards, around-dispatch, and post-execute | `ToolRuntime.execute`, [`tools/src/index.ts`](../packages/core/tools/src/index.ts) and [tool-execution-pipeline.md](tool-execution-pipeline.md) |
| 14 | Result context lands in the next-step list, then `agent/turn-stopping` runs | `ReactLoopAgent.turn`, [`agent-loop/src/agent.ts`](../packages/core/agent-loop/src/agent.ts) |
| 15 | `turn/end` is appended, and the checkpoint policy owns durability | [`session-checkpoint-policy`](../packages/session/session-checkpoint-policy) |

## How context is built and reduced

Anything reaching a model request is reconstructable from the log, so every stage below both reads and writes session events rather than a private buffer.

| Stage | Source | Reference |
|---|---|---|
| Which events reach the model, and how a range is shadowed | [`session/src/surface.ts`](../packages/core/session/src/surface.ts) | [session.md](subsystems/session.md) |
| The workspace instruction chain and its later transitions | [`context/agent-instructions`](../packages/context/agent-instructions) | [its README](../packages/context/agent-instructions/README.md) |
| Skill summaries in the catalog, skill bodies on demand | [`skill/tool-skill`](../packages/skill/tool-skill) | [skills.md](subsystems/skills.md) |
| Request pressure and current surface pricing | [`llm/token-meter`](../packages/llm/token-meter) | [token-meter.md](subsystems/token-meter.md) |
| Model-free tool-result pruning before any summary | [`compaction/compaction-tool-result-pruner`](../packages/compaction/compaction-tool-result-pruner) | [compaction.md](subsystems/compaction.md) |
| Range selection, summarization, and the surface replacement | [`compaction/compaction-basic`](../packages/compaction/compaction-basic) | [compaction.md](subsystems/compaction.md) |
| Oversized tool text stored behind a model-facing locator | [`spill/spill`](../packages/spill/spill) | [spill.md](subsystems/spill.md) |
| Durable storage, resume, and crash recovery | [`session/session-persistence`](../packages/session/session-persistence) | [persistence.md](subsystems/persistence.md) |

## Where to go next

New behavior attaches to an extension point rather than to these files: [architecture.md](architecture.md#where-new-behavior-goes) maps each goal to its mechanism, and the [extension cookbook](cookbook/extension-cookbook.md) indexes the step-by-step guides for [packages](cookbook/adding-a-package.md) and [tools](cookbook/adding-a-tool.md).
