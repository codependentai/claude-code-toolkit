---
skill: cc-architecture
description: "Claude Code internal architecture reference. Use when building plugins, agents, skills, hooks, or debugging CC behavior. Covers boot sequence, query loop, tool system, permissions, context management, and extension points."
trigger: "when building or debugging Claude Code extensions (plugins, agents, skills, hooks), when asking how CC works internally, when needing to understand the tool system, permission model, boot sequence, or query loop"
---

# Claude Code Architecture Reference

Source: verified against Claude Code source (March 2026). Numbers and structure may shift between releases.

## Codebase Scale

| Metric | Count |
|--------|-------|
| Source files | ~1,928 |
| Lines of TypeScript | ~121K |
| Built-in tools | 46 |
| CLI commands | 88 |
| Service modules | 22 |
| React hooks | 104 |
| UI components | 392 files |
| Feature flags | 88-99 |

## Source Organization

- **Entrypoints:** `cli.tsx` (bootstrap), `init.ts`, `mcp.ts` (MCP server mode), `sdk/` (Agent SDK types + schemas)
- **Commands:** 88 dirs — commit, config, memory, doctor, upgrade, login, daemon, bridge, bg, etc.
- **Tools:** 46 tools — BashTool, FileReadTool, FileEditTool, AgentTool, MCPTool, WebFetchTool, Glob, Grep, Write, TodoWrite, ToolSearch, SkillTool, WorkflowTool, TungstenTool, TeamCreateTool, TeamDeleteTool, etc.
- **Services:** 22 dirs — `api/` (Claude API client), `mcp/` (protocol, 24 files), `tools/` (StreamingToolExecutor, registry), `compact/` (context compaction), `analytics/`, `oauth/`, `plugins/`
- **UI:** `components/` (392 files, Ink terminal UI), `hooks/` (104 files, React hooks), `ink/` (custom reconciler + Yoga flexbox + ANSI parser)
- **Utilities:** `permissions/` (24 files), `bash/` (AST parser for safety), `model/` (selection/routing), `settings/`, `git.ts`
- **State/Infra:** `state/`, `bridge/` (31 files, IDE remote-control), `cli/` (WS/SSE transport), `memdir/` (8 files, memory persistence), `tasks/` (background task management), `coordinator/` (multi-agent orchestration), `assistant/` (session management), `voice/` (push-to-talk input)

## Boot Sequence

Traced from `entrypoints/cli.tsx` with `profileCheckpoint` markers:

1. **cli_entry** — entry validation, minimal imports
2. **Feature gates evaluated** — compile-time `bun:bundle` tree-shaking
3. **Fast paths (zero overhead, exit early):**
   - `--version` — immediate console.log, no imports
   - `--dump-system-prompt` — load config, model, system prompt, output, exit
   - `--claude-in-chrome-mcp` — load MCP server for Chrome extension
   - `--chrome-native-host` — Chrome native messaging
   - `--computer-use-mcp` (CHICAGO_MCP gate) — computer use MCP
4. **Daemon worker path** (`--daemon-worker=<kind>`) — lean, no configs loaded
5. **Bridge path** (`--bridge`) — enableConfigs, auth check, policy limits, bridge/main.ts
6. **Daemon path** (`daemon` command) — enableConfigs, sinks, daemon/main.ts
7. **Background path** (`bg` command) — enableConfigs, background runner
8. **Template/environment/runner paths** — feature-gated experimental
9. **Main REPL path** — full UI initialization:
   - Config loaded (settings.json, CLAUDE.md files from project tree, .envrc via direnv)
   - Auth checked (OAuth from `~/.claude/auth.json` or `ANTHROPIC_API_KEY`)
   - GrowthBook initialized (remote feature flags async)
   - Tools assembled (46 built-in + MCP tool schemas merged dynamically)
   - MCP servers connected (stdio/SSE/WebSocket in parallel)
   - System prompt built (from 10+ sources: role definition, tool schemas, CLAUDE.md, memory, git context, OS info)
   - Ink renders terminal UI via custom React reconciler + Yoga flexbox
   - Query loop begins, background tasks (MCP heartbeats, analytics) on timers

## Query Loop (Core Cycle)

```
User input -> createUserMessage() -> append to history -> build system prompt
  -> stream to Claude API -> parse response tokens as they arrive
  -> if tool_use blocks: findToolByName() -> canUseTool() -> StreamingToolExecutor
  -> collect tool results -> loop back to API with results
  -> when no tool_use blocks: display final response -> post-sampling hooks -> wait
```

**Key behaviors:**
- **Streaming** — tokens render as they arrive, no waiting for full response
- **Parallel tool execution** — multiple tool calls run concurrently via StreamingToolExecutor
- **Agentic loop** — tool use -> result -> next API call cycles automatically until no tool_use blocks or stop condition
- **Interrupt** — Ctrl+C aborts executor, sends cancellation, preserves conversation history
- **Post-sampling hooks** — after each assistant turn: compaction threshold check, memory extraction, optional dream consolidation
- **Max turns** — configurable limit on agentic loop iterations

## Tool System (46 Built-in)

### Categories

**File Operations:** Read, Edit, Write, Glob, Grep, NotebookEdit
**Execution:** Bash, PowerShell, REPL
**Search & Fetch:** WebFetch, WebSearch, ToolSearch
**Agents:** Agent, SendMessage, TaskCreate/Get/List/Update/Stop/Output
**Teams:** TeamCreate, TeamDelete (coordinator mode)
**Planning:** EnterPlanMode, ExitPlanMode, EnterWorktree, ExitWorktree, VerifyPlanExecution
**MCP:** MCPTool, ListMcpResources, ReadMcpResource, McpAuth
**System:** AskUserQuestion, TodoWrite, Skill, RemoteTrigger, ScheduleCron, Config, Sleep, Brief, SyntheticOutput
**Experimental:** Tungsten, Workflow, LSP

### Feature-Gated (GrowthBook)

Tools that compile in but only surface when flags are enabled: Sleep, Brief, WebBrowser, TerminalCapture, Monitor, Workflow, CtxInspect, Snip, Overflow, Test, Verify, PlanExecution, ListPeers, Tungsten.

## Permission System (Multi-Layered)

More complex than a simple 3-layer model. Key components:

**Permission Modes (5):** default, auto, sandbox, strict, bubble

**Layer 1 — Tool Registry Filter:** `filterToolsByDenyRules()` removes denied tools before Claude sees them.

**Layer 2 — Per-call Check:** `canUseTool()` on each invocation. Checks allow/deny rules against tool name, arguments, working directory.

**Layer 3 — Classifier-Assisted Decisions:**
- `bashClassifier` (BASH_CLASSIFIER flag) — ML-assisted bash command classification
- `yoloClassifier` — auto-mode intelligent permission decisions

**Layer 4 — Interactive Prompt:** No matching rule -> user asked (allow once / allow always / deny). "Allow always" writes to settings.

**Layer 5 — Path Validation:** `filesystem.ts` (64KB) — complex path sandboxing, shadow rule detection.

### Bash Safety (AST-level)

Full shell AST parser in `utils/bash/` — runs BEFORE `canUseTool()`. Flags: `rm -rf /`, fork bombs, `curl | bash`, sudo escalation, tty injection, history manipulation. Flagged = rejected regardless of permission rules.

### Settings.json Rules

```json
{
  "permissions": {
    "allow": ["Bash(git *)"],
    "deny": ["Bash(rm *)", "Write"]
  }
}
```

## Context Management

**Context window:** 200K default, **1M via beta header** (`CONTEXT_1M_BETA_HEADER` in `constants/betas.ts`). Models supporting 1M context get the extended window automatically.

**Tool results limit:** 200K chars per message (`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`).

**Auto-compact** triggers at ~80% usage:
- Old messages above compact boundary summarized by Claude itself
- `CompactBoundaryMessage` marker inserted
- ~40-60% freed, conversation continues seamlessly
- Manual trigger: `/compact`
- Custom compaction instructions configurable in settings
- Additional features: CACHED_MICROCOMPACT, COMPACTION_REMINDERS

## Major Subsystems

### Coordinator (Multi-Agent Orchestration)
`coordinator/` — `coordinatorMode.ts` (19KB). Feature-gated via COORDINATOR_MODE flag. Enables multi-agent orchestration with:
- `isCoordinatorMode()` via `CLAUDE_CODE_COORDINATOR_MODE` env var
- Allowed tools for async agents (ASYNC_AGENT_ALLOWED_TOOLS)
- Scratchpad support (tengu_scratch gate)
- Session mode matching/switching
- Internal worker tools: TeamCreate, TeamDelete, SendMessage, SyntheticOutput

### Bridge (IDE Remote Control)
`bridge/` — 31 files. Key components:
- `bridgeMain.ts` (118KB) — main bridge implementation
- `replBridge.ts` (102KB) — REPL bridge
- `remoteBridgeCore.ts` (40KB) — remote orchestration
- JWT utilities, trusted device management, session runner
- Supports VS Code and JetBrains IDE integrations

### Voice Mode
`voice/` — push-to-talk voice input. GrowthBook kill-switch: `tengu_amber_quartz_disabled`. Requires Anthropic OAuth (not API keys) + native audio module OR SoX fallback. Default in build pipeline (VOICE_MODE flag enabled).

### Memory System (memdir)
`memdir/` — 8 files (~81KB). Persistent memory/context management:
- `memdir.ts` — core memory CRUD
- `memoryTypes.ts` — type definitions (user, feedback, project, reference memories)
- `findRelevantMemories.ts` — relevance-based retrieval
- `teamMemPaths.ts` — team-shared memory paths
- Located at `~/.claude/projects/<project>/memory/`

### Background Tasks
`tasks/` — async task spawning and execution:
- `LocalMainSessionTask` — main session task
- `LocalAgentTask`, `LocalShellTask` — local execution
- `RemoteAgentTask` — remote task execution
- `DreamTask` — post-session consolidation
- `InProcessTeammateTask` — teammate integration

### Assistant (Session Management)
`assistant/` — session selection and history:
- `AssistantSessionChooser.tsx` — session selection UI
- `sessionHistory.ts` — history API (fetchLatestEvents, pagination)

## Extension Points

### MCP Servers
Unlimited tools via Model Context Protocol. Transports: stdio (subprocess), SSE (HTTP streaming), WebSocket. Capabilities: tools, resources, prompts.

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "transport": "sse"
    }
  }
}
```

### Custom Agents
Markdown files in `~/.claude/agents/` — own system prompts, model overrides, tool restrictions. Invoked via Agent tool with `subagent_type`.

### Skills
Markdown files in `~/.claude/skills/` — reusable slash commands with frontmatter triggers. Support parameters, compose with other skills. Auto-discovered at startup.

### Hooks
Shell commands before/after tool execution. Configured per-tool in `settings.json`. Events: PreToolUse, PostToolUse, Stop, Notification. Use cases: linting, logging, notifications, validation.

### Plugins
Community plugins via marketplace. `/plugin install @author/plugin-name`. Can add tools, commands, agents, skills, hooks, MCP servers.

### CLAUDE.md
Project-level instructions. Discovered by walking up directory tree. Injected into system prompt. Support `@import` for modular instruction sets.

## Feature Flags (88 Total)

As of March 2026. 54 compile cleanly, 34 still have gaps. "Compiles" does not mean "ships to users" — most are behind GrowthBook remote gates that Anthropic controls per-account.

### Default / Shipped
These are in the standard build or broadly rolled out:

| Flag | What it does |
|------|-------------|
| VOICE_MODE | Push-to-talk input, `/voice`, dictation UI. Requires OAuth + audio backend (SoX or native) |

### Working Experimental (compile clean, feature-gated)

**Interaction & UI:**

| Flag | What it does |
|------|-------------|
| AWAY_SUMMARY | AFK summary behavior in REPL |
| HISTORY_PICKER | Interactive prompt history picker |
| HOOK_PROMPTS | Pass prompt text into hook execution |
| KAIROS_BRIEF | Brief-only transcript layout + BriefTool UX |
| KAIROS_CHANNELS | Channel notices and MCP channel messaging |
| LODESTONE | Deep-link / protocol registration flows |
| MESSAGE_ACTIONS | Message action entrypoints in UI |
| NEW_INIT | Newer `/init` decision path |
| QUICK_SEARCH | Prompt quick-search |
| SHOT_STATS | Shot-distribution stats views |
| TOKEN_BUDGET | Token budget tracking, warnings, prompt triggers |
| ULTRAPLAN | `/ultraplan` — remote multi-agent planning (Opus-class) |
| ULTRATHINK | Deep thinking mode boost |

**Agent, Memory & Planning:**

| Flag | What it does |
|------|-------------|
| AGENT_MEMORY_SNAPSHOT | Extra custom-agent memory snapshot state |
| AGENT_TRIGGERS | Local cron/trigger tools + bundled trigger skills |
| AGENT_TRIGGERS_REMOTE | Remote trigger tool path |
| BUILTIN_EXPLORE_PLAN_AGENTS | Built-in explore/plan agent presets |
| CACHED_MICROCOMPACT | Cached microcompact through query/API flows |
| COMPACTION_REMINDERS | Reminder copy around compaction/attachment |
| EXTRACT_MEMORIES | Post-query memory extraction hooks |
| PROMPT_CACHE_BREAK_DETECTION | Cache-break detection around compaction |
| TEAMMEM | Team-memory files, watcher hooks, UI messages |
| VERIFICATION_AGENT | Verification-agent guidance in prompts/task tooling |

**Tools, Permissions & Remote:**

| Flag | What it does |
|------|-------------|
| BASH_CLASSIFIER | ML-assisted bash permission decisions |
| BRIDGE_MODE | Remote Control / REPL bridge (IDE integration) |
| CCR_AUTO_CONNECT | CCR auto-connect default path |
| CCR_MIRROR | Outbound-only CCR mirror sessions |
| CCR_REMOTE_SETUP | Remote setup command path |
| CHICAGO_MCP | Computer-use MCP integration (requires `@ant/computer-use-*`) |
| CONNECTOR_TEXT | Connector-text block handling in API/logging/UI |
| MCP_RICH_OUTPUT | Richer MCP UI rendering |
| NATIVE_CLIPBOARD_IMAGE | macOS clipboard image fast path (needs native module) |
| POWERSHELL_AUTO_MODE | PowerShell-specific auto-mode permissions |
| TREE_SITTER_BASH | Tree-sitter bash parser backend |
| TREE_SITTER_BASH_SHADOW | Tree-sitter bash shadow rollout |
| UNATTENDED_RETRY | Unattended retry in API flows |

### Plumbing/Rollout Flags (compile clean, not user-facing)
ABLATION_BASELINE, ALLOW_TEST_VERSIONS, ANTI_DISTILLATION_CC, BREAK_CACHE_COMMAND, COWORKER_TYPE_TELEMETRY, DOWNLOAD_USER_SETTINGS, DUMP_SYSTEM_PROMPT, FILE_PERSISTENCE, HARD_FAIL, IS_LIBC_GLIBC, IS_LIBC_MUSL, NATIVE_CLIENT_ATTESTATION, PERFETTO_TRACING, SKILL_IMPROVEMENT, SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED, SLOW_OPERATION_LOGGING, UPLOAD_USER_SETTINGS

### Broken — Easy Reconstruction (single missing file/asset)
AUTO_THEME, BG_SESSIONS, BUDDY, BUILDING_CLAUDE_APPS, COMMIT_ATTRIBUTION, FORK_SUBAGENT, HISTORY_SNIP, KAIROS_GITHUB_WEBHOOKS, KAIROS_PUSH_NOTIFICATION, MCP_SKILLS, MEMORY_SHAPE_TELEMETRY, OVERFLOW_TEST_TOOL, RUN_SKILL_GENERATOR, TEMPLATES, TORCH, TRANSCRIPT_CLASSIFIER

### Broken — Medium Gaps (partial wiring, missing implementations)
BYOC_ENVIRONMENT_RUNNER, CONTEXT_COLLAPSE, COORDINATOR_MODE (worker agent), DAEMON, DIRECT_CONNECT, EXPERIMENTAL_SKILL_SEARCH, MONITOR_TOOL, REACTIVE_COMPACT, REVIEW_ARTIFACT, SELF_HOSTED_RUNNER, SSH_REMOTE, TERMINAL_PANEL, UDS_INBOX, WEB_BROWSER_TOOL, WORKFLOW_SCRIPTS

### Broken — Large Missing Subsystems
| Flag | Missing | What it would do |
|------|---------|-----------------|
| KAIROS | assistant stack | Persistent assistant mode with session history |
| KAIROS_DREAM | dream task system | Post-session consolidation/reflection |
| PROACTIVE | proactive task/tool stack | Proactive companion behaviors |
