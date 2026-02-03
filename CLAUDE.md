# CLAUDE.md - AI Assistant Guide for Oh My OpenCode

## Project Overview

**Oh My OpenCode** is a batteries-included OpenCode plugin providing multi-model AI agent orchestration, parallel background agents, and crafted LSP/AST tools. Think of it as "oh-my-zsh for OpenCode."

- **Version**: 3.2.2
- **License**: SUL-1.0
- **Author**: YeonGyu-Kim
- **Primary Language**: TypeScript (ESM)
- **Package Manager**: Bun (exclusively)
- **Runtime**: Bun + OpenCode >= 1.0.150

## Quick Reference

### Essential Commands

```bash
bun install          # Install dependencies (NEVER use npm/yarn)
bun run typecheck    # Type check only
bun run build        # Full build: ESM + TypeScript declarations + JSON schema
bun run rebuild      # Clean + build
bun test             # Run all tests (100 test files)
bun run build:schema # Rebuild config schema after modifying src/config/schema.ts
```

### Testing Your Changes

```bash
# 1. Build
bun run build

# 2. Update opencode config to use local build:
#    "plugin": ["file:///path/to/oh-my-opencode/dist/index.js"]

# 3. Restart OpenCode
```

---

## Critical Rules

### Git Workflow

```
master (deployed/published)
   ↑
  dev (integration branch)
   ↑
feature branches (your work)
```

| Rule | Description |
|------|-------------|
| **ALL PRs → `dev`** | Every pull request MUST target the `dev` branch |
| **NEVER PR → `master`** | PRs to `master` are automatically rejected by CI |
| **Never `bun publish`** | Publishing is GitHub Actions only via workflow_dispatch |
| **Never bump version locally** | CI manages versioning |

### Language Policy

**All project communications MUST be in English:**
- Issues, PRs, commit messages, code comments, documentation, AGENTS.md files

### Bun Exclusively

| Do | Don't |
|----|-------|
| `bun install` | `npm install` / `yarn install` |
| `bun run build` | `npm run build` |
| `bun-types` | `@types/node` |
| `bun test` | `jest` / `vitest` |

---

## Architecture

### Directory Structure

```
oh-my-opencode/
├── src/
│   ├── agents/        # 11 AI agents (Sisyphus, Oracle, Librarian, etc.)
│   ├── hooks/         # 34 lifecycle hooks
│   ├── tools/         # 20+ tools (LSP, AST-Grep, delegation, etc.)
│   ├── features/      # Background agents, Claude Code compat, skills
│   ├── shared/        # 66 cross-cutting utilities
│   ├── cli/           # CLI: install, doctor, run commands
│   ├── mcp/           # Built-in MCPs: websearch, context7, grep_app
│   ├── config/        # Zod schema and TypeScript types
│   └── index.ts       # Main plugin entry point
├── packages/          # 11 platform-specific binaries
├── script/            # Build utilities
├── docs/              # Documentation
└── dist/              # Build output (ESM + .d.ts)
```

### Where to Look

| Task | Location | Notes |
|------|----------|-------|
| Add agent | `src/agents/` | Create .ts with factory, add to `agentSources` in utils.ts |
| Add hook | `src/hooks/` | Create directory with `createXXXHook()`, register in index.ts |
| Add tool | `src/tools/` | Directory with index/types/constants/tools.ts |
| Add MCP | `src/mcp/` | Create config, add to index.ts |
| Add skill | `src/features/builtin-skills/` | Create directory with SKILL.md |
| Add command | `src/features/builtin-commands/` | Add template + register in commands.ts |
| Config schema | `src/config/schema.ts` | Zod schema, then `bun run build:schema` |

### Complexity Hotspots

| File | Lines | Purpose |
|------|-------|---------|
| `src/features/builtin-skills/skills.ts` | 1729 | Skill definitions |
| `src/features/background-agent/manager.ts` | 1418 | Task lifecycle, concurrency |
| `src/agents/prometheus-prompt.ts` | 1283 | Planning agent prompt |
| `src/tools/delegate-task/tools.ts` | 1135 | Category-based delegation |
| `src/index.ts` | 788 | Main plugin entry |
| `src/hooks/atlas/index.ts` | 757 | Orchestrator hook |

---

## Code Conventions

### Naming

| Type | Convention | Example |
|------|------------|---------|
| Directories | kebab-case | `delegate-task/`, `claude-code-hooks/` |
| Tool names | snake_case | `lsp_goto_definition`, `delegate_task` |
| Functions | camelCase | `createDelegateTask()` |
| Hook factories | `createXXXHook` | `createThinkModeHook()` |
| Tool factories | `createXXXTool` | `createBackgroundTaskTool()` |

### File Structure for Tools

```
src/tools/[tool-name]/
├── index.ts      # Barrel export
├── tools.ts      # ToolDefinition or factory
├── types.ts      # Zod schemas
└── constants.ts  # Fixed values
```

### File Structure for Hooks

```
src/hooks/[hook-name]/
├── index.ts      # createXXXHook(ctx) function
└── types.ts      # TypeScript interfaces (optional)
```

### Export Pattern

All modules use barrel exports via `index.ts`:
```typescript
// src/tools/index.ts
export * from "./grep";
export * from "./glob";
export * from "./lsp";
```

### TypeScript

- Strict mode enabled
- Use `bun-types` for type definitions
- Never use `as any`, `@ts-ignore`, or `@ts-expect-error`
- All tool/hook inputs validated with Zod schemas

---

## Agent System

### Agent Models

| Agent | Model | Purpose |
|-------|-------|---------|
| **Sisyphus** | claude-opus-4-5 | Primary orchestrator (fallback: kimi-k2.5 → glm-4.7 → gpt-5.2-codex) |
| **Hephaestus** | gpt-5.2-codex | Autonomous deep worker ("The Legitimate Craftsman") |
| **Atlas** | claude-sonnet-4-5 | Master orchestrator |
| **oracle** | gpt-5.2 | Consultation, debugging (read-only) |
| **librarian** | glm-4.7 | Docs, GitHub search |
| **explore** | grok-code-fast-1 | Fast codebase grep |
| **multimodal-looker** | gemini-3-flash | PDF/image analysis |
| **Prometheus** | claude-opus-4-5 | Strategic planning |

### Tool Restrictions

| Agent | Cannot Use |
|-------|-----------|
| oracle | write, edit, task, delegate_task |
| librarian | write, edit, task, delegate_task, call_omo_agent |
| explore | write, edit, task, delegate_task, call_omo_agent |
| multimodal-looker | Everything except: read, glob, grep |

### Adding an Agent

1. Create `src/agents/my-agent.ts` with factory + metadata
2. Add to `agentSources` in `src/agents/utils.ts`
3. Update `AgentNameSchema` in `src/config/schema.ts`
4. Run `bun run build:schema`

```typescript
// src/agents/my-agent.ts
import type { AgentConfig } from "./types";

export const myAgentMetadata = {
  category: "utility",
  cost: "low",
  triggers: ["keyword"],
};

export function createMyAgent(model: string): AgentConfig {
  return {
    name: "my-agent",
    model,
    description: "What this agent does",
    prompt: `Your agent's system prompt`,
    temperature: 0.1, // Max 0.3 for code agents
  };
}
```

---

## Hook System

### Hook Events

| Event | Timing | Can Block | Use Case |
|-------|--------|-----------|----------|
| `UserPromptSubmit` | `chat.message` | Yes | Keyword detection, slash commands |
| `PreToolUse` | `tool.execute.before` | Yes | Validate/modify inputs, inject context |
| `PostToolUse` | `tool.execute.after` | No | Truncate output, error recovery |
| `Stop` | `session.stop` | No | Auto-continue, notifications |
| `onSummarize` | Compaction | No | Preserve state |

### Adding a Hook

1. Create `src/hooks/my-hook/index.ts`
2. Export `createMyHook(ctx)` returning event handlers
3. Add hook name to `HookNameSchema` in `src/config/schema.ts`
4. Register in `src/index.ts`

```typescript
// src/hooks/my-hook/index.ts
import type { PluginInput } from "@opencode-ai/plugin";

export function createMyHook(input: PluginInput) {
  return {
    "tool.execute.after": async (toolInput, output) => {
      // Your logic here
    },
  };
}
```

---

## Tool System

### Tool Categories

| Category | Tools | Pattern |
|----------|-------|---------|
| LSP | lsp_goto_definition, lsp_find_references, lsp_symbols, lsp_diagnostics, lsp_prepare_rename, lsp_rename | Direct |
| Search | ast_grep_search, ast_grep_replace, grep, glob | Direct |
| Session | session_list, session_read, session_search, session_info | Direct |
| Task | task_create, task_get, task_list, task_update | Factory |
| Agent | delegate_task, call_omo_agent | Factory |
| Background | background_output, background_cancel | Factory |

### Tool Patterns

**Direct ToolDefinition** (static):
```typescript
export const grep: ToolDefinition = tool({
  description: "Search files",
  args: { pattern: tool.schema.string() },
  execute: async (args) => result,
});
```

**Factory Function** (context-dependent):
```typescript
export function createDelegateTask(ctx, manager): ToolDefinition {
  return tool({ execute: async (args) => { /* uses ctx */ } });
}
```

---

## MCP Architecture

Three-tier system:

1. **Built-in** (`src/mcp/`): websearch (Exa), context7 (docs), grep_app (GitHub)
2. **Claude Code compat**: `.mcp.json` with `${VAR}` expansion
3. **Skill-embedded**: YAML frontmatter in skills

### Adding an MCP

1. Create `src/mcp/my-mcp.ts`
2. Add to `allBuiltinMcps` in `src/mcp/index.ts`
3. Add to `McpNameSchema` in `src/mcp/types.ts`

```typescript
// src/mcp/my-mcp.ts
export const my_mcp = {
  type: "remote" as const,
  url: "https://...",
  enabled: true,
  oauth: false as const,
};
```

---

## Testing

### TDD Required

**RED-GREEN-REFACTOR**:
1. **RED**: Write test → `bun test` → FAIL
2. **GREEN**: Implement minimum → PASS
3. **REFACTOR**: Clean up → stay GREEN

### Test Conventions

- Test files: `*.test.ts` alongside source
- BDD comments: `//#given`, `//#when`, `//#then`
- 100 test files in the codebase

### Running Tests

```bash
bun test                    # All tests
bun test src/hooks/         # Tests in specific directory
bun test my-feature.test.ts # Single test file
```

---

## Anti-Patterns (Don't Do)

| Category | Forbidden |
|----------|-----------|
| Package Manager | npm, yarn - Bun exclusively |
| Types | @types/node - use bun-types |
| Type Safety | `as any`, `@ts-ignore`, `@ts-expect-error` |
| Publishing | Direct `bun publish` - GitHub Actions only |
| Versioning | Local version bump - CI manages |
| File Ops | mkdir/touch/rm in code - use bash tool |
| Error Handling | Empty catch blocks |
| Testing | Deleting failing tests, writing implementation before test |
| Agent Calls | Sequential calls - use `delegate_task` parallel with `run_in_background` |
| Hook Logic | Heavy computation in PreToolUse - slows every call |
| Commits | Giant commits (3+ files), separate test from implementation |
| Temperature | > 0.3 for code agents |
| Trust | Agent self-reports - ALWAYS verify outputs |
| Git Interactive | `git add -i`, `git rebase -i` (no interactive input) |
| Git Hooks | Skip hooks (--no-verify), force push without request |
| Bash | `sleep N` - use conditional waits |
| Bash | `cd dir && cmd` - use workdir parameter |

---

## OpenCode Plugin Development

### Accessing OpenCode Source

When you need to examine OpenCode internals:

```bash
git clone https://github.com/sst/opencode /tmp/opencode-source
```

Use the **librarian agent** for plugin work:
- Searching OpenCode hook implementations
- Finding OpenCode tool patterns
- Examining OpenCode SDK source
- Understanding "how does OpenCode do X?"

**DO NOT guess about OpenCode internals** - always verify by examining source.

---

## Configuration

### Config Locations (Priority Order)

1. `.opencode/oh-my-opencode.json` (project)
2. `~/.config/opencode/oh-my-opencode.json` (user)

### JSONC Support

- Line comments: `// comment`
- Block comments: `/* comment */`
- Trailing commas allowed

### Schema Autocomplete

```json
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/master/assets/oh-my-opencode.schema.json"
}
```

---

## Useful Utilities

Import from `src/shared`:

| Utility | Purpose |
|---------|---------|
| `logger.ts` | File-based logging (`/tmp/oh-my-opencode.log`) |
| `dynamic-truncator.ts` | Token-aware context window management |
| `model-resolver.ts` | 3-step model resolution |
| `jsonc-parser.ts` | JSONC parsing with comments |
| `frontmatter.ts` | YAML frontmatter extraction |
| `data-path.ts` | XDG-compliant storage resolution |
| `permission-compat.ts` | Agent tool restriction enforcement |
| `system-directive.ts` | System message prefix and filtering |
| `deep-merge.ts` | Recursive object merging (proto-pollution safe) |

---

## Deployment

**GitHub Actions workflow_dispatch ONLY**:

1. Commit & push changes to `dev` branch
2. Create PR targeting `dev`
3. After merge, maintainers trigger: `gh workflow run publish -f bump=patch`

**Never**:
- Run `bun publish` directly
- Bump version in package.json locally
- Push directly to master

---

## Getting Help

- **Project Knowledge**: Check AGENTS.md files in subdirectories
- **Code Patterns**: Review existing implementations in `src/`
- **Issues**: https://github.com/code-yeongyu/oh-my-opencode/issues
- **Discord**: https://discord.gg/PUwSMR9XNk

---

*This file was generated for AI assistants working on the oh-my-opencode codebase. Last updated: 2026-02-03.*
