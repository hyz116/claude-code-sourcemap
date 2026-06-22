# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an **unofficial research-only** reconstruction of `@anthropic-ai/claude-code` v2.1.88, extracted from the npm package's source map (`cli.js.map` → `sourcesContent`). It contains 4,756 restored files (1,884 TypeScript/TSX sources, ~512K LoC). Copyright belongs to Anthropic. Do not use commercially.

The restored source lives in `restored-src/src/`. The `docs/` directory contains architecture analysis in Chinese.

## Research Synthesis Discipline

When writing wiki concept pages / 设计原则 / insight 文档 / 综合分析 in `docs/`, follow the bias checklist in auto-memory `research-synthesis-bias-checklist` (loaded via MEMORY.md). Specifically: label claims as OBS/INF/SPEC/NARR; check the 9 induction biases before each section; apply the "+1 怀疑度 per abstraction level" principle.

Does **not** apply to: code editing, debugging, descriptive code documentation, or skill executions.

The full self-critique record (4 rounds) is in `docs/insights/3.SELF_CRITIQUE.md`.

## No Build/Test Commands

This repo has no build pipeline — the source is a read-only research artifact. The only script in `package.json` is a guard that prevents unauthorized publishing. There are no runnable tests in this repo.

The extraction script is `extract-sources.js`, which reads `package/cli.js.map` and writes restored source files.

## Tech Stack

- **Runtime**: Bun (fast startup, native TS, DCE via `feature()`)
- **Language**: TypeScript + TSX
- **TUI**: React + Ink (terminal UI)
- **API**: `@anthropic-ai/sdk` with Zod v4 validation
- **CLI**: Commander.js

## Architecture

### Core Agent Loop (`query.ts` + `QueryEngine.ts`)

`QueryEngine.ts` is the high-level session coordinator — assembles system prompts, normalizes messages, filters tool availability, persists state. It delegates to `query.ts` which implements the actual agent iteration as an AsyncGenerator:

1. Build API request → stream Anthropic API → parse `tool_use` blocks → execute tools → collect `tool_result` → loop until `end_turn`

Message compression happens inside this loop in four escalating stages: **Snip** (drop old snippets) → **Microcompact** (merge adjacent tool pairs) → **Context Collapse** (fold history segments) → **Autocompact** (call Claude to summarize).

### Tool System (`Tool.ts` + `tools/`)

Every tool implements a contract:
- `call(input, context)` → `AsyncGenerator<StreamEvent>` (streaming progress, not callbacks)
- `isReadOnly(input)` → boolean (Plan Mode filter)
- `isConcurrencySafe()` → controls parallel execution

`ToolUseContext` acts as a dependency-injection container passed through the entire call chain. Tools with large outputs spill results to disk (`maxResultSizeChars`). MCP tools are dynamically proxied at runtime as servers connect.

40+ built-in tools span: file ops (Read/Edit/Write/Glob/Grep), execution (Bash), agents (Agent/SendMessage/Task\*), search (WebFetch/WebSearch/ToolSearch), planning (EnterPlanMode), worktrees (EnterWorktree), and MCP proxies.

### Multi-Agent System (`AgentTool` + `tasks/`)

`AgentTool` is the unified entry point for spawning subagents with four execution modes:
1. **Sync** — blocks current turn, result returned inline
2. **Async background** — returns `agentId`, result arrives via `task-notification` XML
3. **Remote CCR** — cloud execution, polled asynchronously
4. **Swarm/Teammate** — multi-agent coordination via tmux/iTerm2 panes

Subagent results are injected back into the parent session as `task-notification` XML messages. In **Coordinator Mode**, the main agent only orchestrates and never executes tools directly.

### State Management (`state/`)

A custom 35-line pub/sub store (`store.ts`) with `getState()` / `setState(prev => next)` / `subscribe(listener)` — no Redux. React integration uses `useSyncExternalStore`. `AppState` has 80+ fields covering: messages, tasks registry, permission context, MCP clients/tools, plugin state, CCR bridge state, and UI state.

Side effects on state changes are centralized in `onChangeAppState.ts` (CCR sync, SDK emit, config persist).

### Permission System (`utils/permissions/`)

Four-layer rule priority (highest to lowest): **MDM** (enterprise) → **Global** `~/.claude/settings.json` → **Project** `.claude/settings.json` → **Local** `.claude/settings.local.json`.

Four permission modes: `default` (prompt for dangerous ops), `auto` (approve most), `bypass` (no prompts), `plan` (read-only). Bash commands additionally go through an AST parser + ML classifier to predict approval before showing a UI dialog.

### Services (`services/`)

| Service | Role |
|---------|------|
| `api/` | Anthropic SDK wrapper, streaming, retry, error handling |
| `mcp/` | MCP client lifecycle, stdio/SSE/HTTP transports |
| `compact/` | Auto-compression when token budget exceeded |
| `tools/StreamingToolExecutor` | Parallel tool execution overlapping model generation |
| `SessionMemory/` | Memory extraction, persistence, retrieval |
| `analytics/` | GrowthBook + Statsig feature flags + telemetry |
| `policyLimits/` | Enterprise quota/rate enforcement |

### Entrypoints

| Mode | File |
|------|------|
| CLI (interactive) | `entrypoints/cli.tsx` |
| SDK (headless) | `entrypoints/sdk/` |
| MCP Server | `entrypoints/mcp.ts` |
| CLI dispatch | `src/main.tsx` |

### Skills System (`skills/`)

Skills can come from three sources: built-in (compiled into the binary), disk (`~/.claude/skills/`), or MCP prompts. Execution is either **inline** (continues in current agent) or **fork** (spawns a child agent). `SkillTool` handles both.
