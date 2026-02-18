# Comparison: How Three Projects Invoke the Claude CLI

Comparing `claude-agent-sdk-python`, `gastown`, and `vibe-kanban` — three different approaches to programmatically driving the `claude` CLI binary.

---

## Spawning

| | **claude-agent-sdk-python** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| **Language** | Python (async) | Go | Rust (async) |
| **Spawn method** | `anyio.open_process()` | `exec env ... claude` via tmux | `tokio::process::Command` |
| **Binary source** | Bundled CLI first, then PATH | `claude` from PATH | `npx -y @anthropic-ai/claude-code@2.1.32` (pinned version via npx) |
| **Process wrapper** | Direct subprocess | tmux session (`gt-<rig>-<polecat>`) | Process group with `kill_on_drop` |

**Key difference:** Gastown runs `claude` inside **tmux sessions** and interacts via `tmux send-keys` / `tmux capture-pane` — it treats Claude as a terminal user. The other two pipe stdin/stdout directly.

---

## Communication Protocol

| | **SDK** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| **Protocol** | Bidirectional JSON-lines (control protocol) | Plain text via tmux | Bidirectional JSON-lines (control protocol) |
| **`--input-format stream-json`** | Yes (hardcoded) | No | Yes |
| **`--output-format stream-json`** | Yes (hardcoded) | No | Yes |
| **Initialize handshake** | Yes (sends `initialize` control request) | No | Yes (sends `initialize` with hooks) |
| **Control requests/responses** | Full support | None | Full support |

**Key difference:** Gastown is the outlier — it uses **no structured protocol at all**. It sends text to a tmux pane and scrapes output. The SDK and vibe-kanban both implement the full bidirectional JSON control protocol with initialize handshakes, control requests, and correlated responses.

---

## Hardcoded Flags

| Flag | **SDK** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| `--output-format stream-json` | Always | Never | Always |
| `--input-format stream-json` | Always | Never | Always |
| `--verbose` | Always | Never | Always |
| `--include-partial-messages` | Optional | No | Always |
| `--replay-user-messages` | No | No | Always |
| `-p` (print/prompt mode) | No | No | Always |
| `--dangerously-skip-permissions` | No | **Always (hardcoded)** | Optional (config) |
| `--system-prompt ""` | **Always (blanks it)** | Never | Never |
| `--setting-sources ""` | **Always (disables settings)** | Never | Never |
| `--disallowedTools AskUserQuestion` | No | No | Always |
| `--permission-prompt-tool stdio` | If callback provided | No | If approvals/plan enabled |

**Key differences:**
- **SDK blanks the system prompt and disables settings** — the other two don't touch these, inheriting Claude Code's defaults
- **Gastown always skips permissions** via `--dangerously-skip-permissions` — simplest approach
- **Vibe-kanban disallows `AskUserQuestion`** since there's no human to answer it

---

## Permissions

| | **SDK** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| **Model** | Callback-based (`can_use_tool`) | Skip all | Three modes: bypass / default / plan |
| **Can rewrite tool inputs** | Yes (`updatedInput`) | No | Yes (via hook callbacks) |
| **Hook support** | Yes (PreToolUse, PostToolUse, Stop, etc.) | Via `.claude/commands/` files only | Yes (PreToolUse, Stop) |
| **Approval service** | User callback function | None | Optional `ExecutorApprovalService` |

---

## Environment Variables

| | **SDK** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| **Identity markers** | Commented out (`CLAUDE_CODE_ENTRYPOINT`) | None | None |
| **Custom env vars** | `options.env` | `GT_ROLE`, `GT_RIG`, `GT_POLECAT`, `GT_ROOT`, `BD_ACTOR`, `GIT_AUTHOR_NAME` | `NPM_CONFIG_LOGLEVEL=error`, optional custom env |
| **API key handling** | Relies on existing auth | Relies on existing auth | Can explicitly remove `ANTHROPIC_API_KEY` (`disable_api_key`) |

**Key difference:** Gastown injects a rich set of **identity/coordination env vars** (`GT_ROLE`, `BD_ACTOR`, `GIT_AUTHOR_NAME`) because it orchestrates multiple agents (polecats) that need to know their role in a team.

---

## Session Management

| | **SDK** | **gastown** | **vibe-kanban** |
|---|---|---|---|
| **Resume support** | `--resume <session_id>` | Tmux session persists naturally | `--resume <session_id>` + `--resume-session-at <msg_id>` |
| **Interactive (multi-turn)** | `ClaudeSDKClient` keeps stdin open | Tmux send-keys for follow-ups | Resume with new messages |
| **MCP servers** | In-process bridge via control protocol | No | No |
| **Mid-session model change** | Yes (`set_model` control) | No | Yes (`setPermissionMode` control) |

---

## Architectural Summary

| Project | Philosophy |
|---|---|
| **claude-agent-sdk-python** | **Full-featured SDK wrapper.** Implements the complete control protocol. Blanks system prompt and settings for isolation. Supports in-process MCP servers, hooks, permission callbacks. Designed as a general-purpose library. |
| **gastown** | **Tmux-based orchestrator.** Treats `claude` as a terminal app. No structured protocol — just sends text and reads panes. Simplest approach but no programmatic control over permissions or tools. Compensates with rich env vars for multi-agent coordination. |
| **vibe-kanban** | **Rust task runner with approval workflows.** Full control protocol like the SDK, but adds task-management-specific features: plan mode, approval services, disallowed tools, slash commands. Pins exact CLI versions via npx. Most opinionated about workflow. |

The spectrum goes: **gastown** (simplest, text-over-tmux) → **SDK** (general-purpose protocol wrapper) → **vibe-kanban** (opinionated task orchestrator with approval workflows).
