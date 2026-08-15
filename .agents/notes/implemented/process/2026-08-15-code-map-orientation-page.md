# Agent Note: Code Map orientation page

Status: implemented

English | [中文](2026-08-15-code-map-orientation-page.zh.md)

## Problem

A reader who is about to change `packages/` can find every fact separately and still not know where to start reading. [architecture.md](../../../../docs/architecture.md) describes turn behavior without naming the functions that implement it, the [subsystems pages](../../../../docs/subsystems/README.md) define types without an ordered reading path, and package READMEs describe one package at a time. Nothing connected the documented turn flow to the source that runs it, so orientation was rediscovered per reader — architecture.md answered it by recommending an agent explore the codebase.

## Decision

[docs/code-map.md](../../../../docs/code-map.md) is the orientation reference: which package owns each concern, the ordered path one turn takes through the source, and the stages that build and reduce model context. architecture.md links to it from its opening.

The page's subject is the mapping itself, so every row links to the owner instead of restating it — the tier taxonomy's one-home rule is what makes the page affordable to keep current. It cites functions and files (`ReactLoopAgent.buildRequest`, [`agent-loop/src/tool-calls.ts`](../../../../packages/core/agent-loop/src/tool-calls.ts)) rather than line numbers, which drift silently and no gate checks.

The page carries no fenced `ts` blocks. Pasted implementation code would duplicate a home, drift from source without a gate catching it, and pull the page into the `type-equiv` manifest for types it does not own.

## Alternatives considered

**Expand architecture.md instead of adding a page.** Its budget has room, but its job is the ordered behavioral map, and source anchors are a different lookup — a reader tracing a bug wants the call chain, not the composition model. Mixing them also puts churn-prone file paths inside the repo's most-read page.

**Publish the module-by-module walkthrough verbatim, with implementation excerpts.** This was the drafted form. It restated `session.md`, `system-prompt.md`, `tools.md`, `compaction.md`, and [tool-execution-pipeline.md](../../../../docs/tool-execution-pipeline.md) at length, which the [slop checklist](../../../../docs/AGENTS.md) rejects as the same fact in more than one home, and its code blocks would have gone stale on the first refactor. The mapping survived; the restatement did not.

**Put it under `docs/user/develop/`.** That tree is the product-facing published guide. A source-anchored reading order is contributor material, and the tier table keeps contributor procedures out of `user/`.

**Generate it from source.** The turn path is an authored reading order, not a derivable relation; a generator would need the same hand-written sequence as input. The graph generators already own what is mechanically derivable ([module-graph.md](../../../../docs/module-graph.md), [agent-lifecycle.md](../../../../docs/agent-lifecycle.md)).

## Consequences

Renaming a core file or driver method makes the page wrong without failing a gate: `verify-md-links` checks the link targets, so a moved file is caught, but a renamed method in prose is not. The page is deliberately short and pointer-shaped to keep that exposure small.

The page is not projected to the documentation website; it is reachable from architecture.md, which the website does project. Adding a route is a [website/docs.ts](../../../../website/docs.ts) mapping change whenever contributor orientation is wanted there.
