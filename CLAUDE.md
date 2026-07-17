# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build all packages (order: tui → ai → agent → coding-agent → orchestrator)
npm run build

# Full check: lint + format + types + pinned deps + shrinkwrap + browser smoke
npm run check

# Run non-e2e tests (stashes auth.json, unsets all API keys first)
./test.sh

# Run pi from sources (works from any directory)
./pi-test.sh

# Run pi without API keys
./pi-test.sh --no-env

# Run a single test file (from package root)
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts

# Regenerate AI model catalog (after editing generate-models.ts, never edit models.generated.ts directly)
npm run generate:models
```

**Never run `npm run build` or `npm test` unless asked.** Use `npm run check` after code changes (not docs). Never run the full vitest suite directly — it triggers e2e tests when API keys are present. Use `./test.sh` instead.

## Architecture

```
packages/
  ai/          @earendil-works/pi-ai        Unified multi-provider LLM API
  agent/       @earendil-works/pi-agent-core Agent runtime (loop, tools, state, compaction)
  coding-agent/ @earendil-works/pi-coding-agent  CLI + TUI coding agent (depends on ai, agent, tui)
  tui/         @earendil-works/pi-tui        Terminal UI library (differential rendering)
  orchestrator/ @earendil-works/pi-orchestrator  Multi-agent orchestration (experimental)
```

Dependencies flow: `ai` ← `agent` ← `coding-agent` ← `orchestrator`. `tui` is used by `coding-agent`.

### `packages/ai` — LLM abstraction

- **`src/types.ts`**: Core types (`Model`, `Api`, `Context`, `Message`, `Provider`).
- **`src/models.ts`**: `Provider` interface — `createProvider()` takes an `id`, model catalog, auth resolver, and stream factory. Model catalogs are auto-generated in `src/models.generated.ts` (never edit directly; edit `scripts/generate-models.ts` instead).
- **`src/api/`**: Per-provider-API stream implementations (anthropic-messages, openai-responses, google-generative-ai, bedrock-converse-stream, etc.). Each has a lazy variant for tree-shaking.
- **`src/providers/`**: Provider factories that wire up APIs with model catalogs (`*.models.ts`) and auth. `faux.ts` is the test provider — deterministic, no network.
- **`src/auth/`**: Credential store, OAuth, API key resolution. `auth.json` in `~/.pi/agent/`.
- **`src/compat/`**: Re-exports for backward compatibility with the old global API surface.

### `packages/agent` — Agent runtime

- **`src/agent.ts`**: Core agent — wraps a transport (`streamSimple`) with tool calling loop, state management, and event emission.
- **`src/agent-loop.ts`**: `runAgentLoop()` / `runAgentLoopContinue()` — the main agent logic: receive messages, call model, execute tools, handle queueing.
- **`src/types.ts`**: `Agent`, `AgentTool`, `AgentState`, `AgentEvent`, `ToolExecutionMode`, `QueueMode`.
- **`src/harness/`**: Higher-level harness built on the core agent — session persistence (`jsonl-repo`/`memory-repo`), compaction, system prompts, skills, prompt templates.

### `packages/coding-agent` — The CLI app

- **`src/cli/`**: CLI entrypoint — `args.ts` (argument parsing), `startup-ui.ts`, session picker, config selector.
- **`src/core/agent-session.ts`**: `AgentSession` — shared by all run modes (interactive, print, RPC). Wraps the agent harness with model management, compaction, bash execution, event bus, and extension lifecycle.
- **`src/core/tools/`**: Built-in tools — `bash.ts`, `edit.ts`, `write.ts`, `read.ts`, `grep.ts`, `find.ts`, `ls.ts`, `edit-diff.ts`.
- **`src/core/extensions/`**: Extension system — `types.ts` defines `Extension`; `runner.ts` loads and executes extensions; `wrapper.ts` wraps tools from extensions with sandboxing.
- **`src/modes/interactive/`**: Interactive TUI mode — components, theme, keybindings.
- **`src/modes/rpc/`**: RPC/headless mode (for IDE integration, orchestrator).
- **`src/modes/print-mode.ts`**: Non-interactive `-p` flag mode.
- **`src/core/compaction/`**: Context window management — summarization, branch summaries, file operation tracking.

### `packages/tui` — Terminal UI

- Differential rendering TUI library. Components: `Editor`, `Markdown`, `SelectList`, `Input`, `Box`, `Loader`, etc.
- Keybinding system (`KeybindingsManager`), fuzzy matching, terminal image support (Kitty/iTerm2 protocols).

## Key conventions

- **Node >= 22.19.0**. TypeScript 5.9 with `erasableSyntaxOnly` — no `enum`, `namespace`, parameter properties, `import =`, or other non-erasable constructs. `moduleResolution: Node16`, ESM only.
- **Biome** for lint + format: tabs, indent 3, line width 120. Config in `biome.json`.
- **Lockstep versioning**: all packages share the same version. `patch` = fixes/additions, `minor` = breaking changes. No major releases.
- **`package-lock.json` is ground truth**: external deps pinned to exact versions. Pre-commit blocks lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`.
- **Install always with `--ignore-scripts`**: `npm install --ignore-scripts`, `npm ci --ignore-scripts`.
- **No inline/dynamic imports** — top-level `import` statements only.
- **Never hardcode key checks** — add to `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS` for configurability.
- **Commit messages**: `{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <description>`.
- **Stage explicit paths**: `git add <path>` — never `git add -A` / `git add .`. Multiple pi sessions may run concurrently in the same cwd.
- **Changelogs**: `packages/*/CHANGELOG.md` under `## [Unreleased]`. Released sections are immutable.

## Testing

- **Faux provider** (`packages/ai/src/providers/faux.ts`): deterministic, no-network provider for all non-e2e tests. Tests using `packages/coding-agent/test/suite/harness.ts` use the faux provider automatically.
- **Regression tests**: `packages/coding-agent/test/suite/regressions/<issue>-<slug>.test.ts`.
- **Test credentials**: `./test.sh` stashes `~/.pi/agent/auth.json` and unsets all API key env vars before running tests.
- **Suite harness**: `packages/coding-agent/test/suite/harness.ts` provides session setup with faux provider — use for all suite tests.

## Git safety

Multiple pi sessions may run concurrently in this repo. Never run destructive commands: `git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`, `git add -A`, `git add .`, `git commit --no-verify`. Stage only your own files with explicit paths. If rebase conflicts occur in files you didn't modify, abort and ask the user.
