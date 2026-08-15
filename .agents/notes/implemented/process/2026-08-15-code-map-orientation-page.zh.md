# Agent Note: Code Map 定位页面

Status: implemented

[English](2026-08-15-code-map-orientation-page.md) | 中文

## 问题

即将改动 `packages/` 的读者可以逐条找到每个事实，却仍然不知道该从哪里开始读。[architecture.md](../../../../docs/architecture.md) 描述轮次行为，但不点名实现它的函数；[各子系统页面](../../../../docs/subsystems/README.md)定义类型，但没有有序的阅读路径；包 README 一次只描述一个包。此前没有任何内容把有文档的轮次流程与运行它的源码连接起来，因此每位读者都要重新摸索一遍定位过程 —— architecture.md 对此的回答是建议用 agent 探索代码库。

## 决策

[docs/code-map.md](../../../../docs/code-map.md) 是定位参考：每项关注点由哪个包拥有、一个轮次在源码中经过的有序路径，以及构建与缩减模型上下文的各个阶段。architecture.md 在开篇处链接到它。

该页的主题就是这份映射本身，因此每一行都链接到其拥有者而不复述其内容 —— 正是 tier 分类法的一处归属规则，使这一页的维护成本可以承受。它引用函数与文件（`ReactLoopAgent.buildRequest`、[`agent-loop/src/tool-calls.ts`](../../../../packages/core/agent-loop/src/tool-calls.ts)），而不是行号：行号会无声漂移，且没有任何 gate 检查它。

该页不含 `ts` 代码围栏。粘贴实现代码会复制一处归属、在没有 gate 捕获的情况下与源码脱节，并且会把这一页拉进它并不拥有的类型的 `type-equiv` 清单。

## 考虑过的替代方案

**扩写 architecture.md 而不新增页面。** 它的预算尚有余量，但它的职责是有序的行为地图，而源码锚点是另一种查阅方式 —— 追查缺陷的读者想要的是调用链，而不是组合模型。把两者混在一起，还会把易变的文件路径塞进本仓库阅读量最大的页面。

**逐字发布逐模块讲解，并附实现摘录。** 这是最初起草的形态。它大段复述了 `session.md`、`system-prompt.md`、`tools.md`、`compaction.md` 与 [tool-execution-pipeline.md](../../../../docs/tool-execution-pipeline.md)，而[水分清单](../../../../docs/AGENTS.md)正是以「同一事实出现在多处归属」为由拒绝这种写法；其代码块也会在第一次重构后过时。留下来的是映射，被去掉的是复述。

**放到 `docs/user/develop/` 下。** 那棵树是面向产品、对外发布的指南。以源码为锚的阅读顺序属于贡献者材料，而 tier 表格将贡献者流程排除在 `user/` 之外。

**由源码生成。** 轮次路径是人工编写的阅读顺序，而非可推导的关系；生成器仍需要同一份手写序列作为输入。可机械推导的部分已由图生成器拥有（[module-graph.md](../../../../docs/module-graph.md)、[agent-lifecycle.md](../../../../docs/agent-lifecycle.md)）。

## 后果

重命名一个核心文件或驱动器方法会让该页出错，却不会让任何 gate 变红：`verify-md-links` 检查链接目标，因此文件移动会被捕获，但正文中被重命名的方法不会。该页刻意保持简短、以指针为主，正是为了把这一暴露面压到最小。

该页不投射到文档网站；它可从 architecture.md 到达，而网站确实投射了后者。若希望在网站上提供贡献者定位内容，添加路由是一处 [website/docs.ts](../../../../website/docs.ts) 映射变更。
