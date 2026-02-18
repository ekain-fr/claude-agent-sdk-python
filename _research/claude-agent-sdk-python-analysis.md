# Deep Analysis: How `claude-agent-sdk-python` Invokes the Claude CLI

## Executive Summary

The SDK is a **Python async wrapper** around the `claude` CLI binary. It spawns `claude` as a subprocess, communicates via **bidirectional JSON-lines on stdin/stdout**, and layers a **control protocol** on top for features like permission callbacks, hooks, and in-process MCP servers. Compared to calling `claude` directly, the SDK imposes several notable differences.

---

## 1. How `claude` Is Called

**File:** `src/claude_agent_sdk/_internal/transport/subprocess_cli.py:372-380`

```python
self._process = await anyio.open_process(
    cmd,
    stdin=PIPE, stdout=PIPE, stderr=stderr_dest,
    cwd=self._cwd,
    env=process_env,
    user=self._options.user,
)
```

Uses `anyio.open_process()` (async subprocess). All communication goes through piped stdin/stdout.

---

## 2. Hardcoded Flags (Always Present)

Every invocation includes these flags — you **cannot** disable them:

```
claude --output-format stream-json --verbose --input-format stream-json
```

| Flag | Effect |
|------|--------|
| `--output-format stream-json` | Forces JSON-lines output (not human-readable text) |
| `--verbose` | Enables verbose logging |
| `--input-format stream-json` | Enables bidirectional JSON-lines on stdin |

Additionally:
- `--system-prompt ""` is **always sent** when no system prompt is provided (line 170-171) — this **overrides** the CLI's default system prompt with an empty one
- `--setting-sources ""` is sent by default when no setting sources are specified (line 276-281) — this **disables** loading of user/project/local settings

These two defaults are a **major behavioral difference** from calling `claude` directly.

---

## 3. Environment Variables

**Set by the SDK (line 349-361):**

| Variable | Value | Condition |
|----------|-------|-----------|
| *(all of `os.environ`)* | inherited | always |
| *(user-provided `options.env`)* | custom | if specified |
| `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` | `"true"` | if `enable_file_checkpointing=True` |
| `PWD` | custom cwd | if `cwd` specified |

**Deliberately COMMENTED OUT (the fork's key change):**

```python
# "CLAUDE_CODE_ENTRYPOINT": "sdk-py",
# "CLAUDE_AGENT_SDK_VERSION": __version__,
```

In the **upstream official SDK**, these are set. They tell the CLI "this is an SDK call", which triggers **API-key-only authentication**. This fork comments them out so the CLI treats it as a direct invocation and accepts **Max subscription auth**.

The same marker is also commented out in:
- `client.py:69` — `CLAUDE_CODE_ENTRYPOINT = "sdk-py-client"`
- `query.py:122` — `CLAUDE_CODE_ENTRYPOINT = "sdk-py"`

**Recognized env vars (read, not set):**
- `CLAUDE_AGENT_SDK_SKIP_VERSION_CHECK` — skip CLI version validation
- `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT` — timeout in ms (default 60000)

---

## 4. CLI Binary Discovery Order

**File:** `subprocess_cli.py:64-95`

1. **Bundled CLI** — `src/claude_agent_sdk/_bundled/claude` (ships with pip package)
2. **`shutil.which("claude")`** — system PATH
3. **Hardcoded fallbacks:**
   - `~/.npm-global/bin/claude`
   - `/usr/local/bin/claude`
   - `~/.local/bin/claude`
   - `~/node_modules/.bin/claude`
   - `~/.yarn/bin/claude`
   - `~/.claude/local/claude`
4. **Explicit override** — `ClaudeAgentOptions(cli_path=...)`

The bundled CLI is preferred, meaning the SDK may use a **different version** than what's on your PATH.

---

## 5. The Control Protocol (Bidirectional JSON on stdin/stdout)

This is the biggest difference from a simple `claude -p "prompt"` invocation. The SDK doesn't just send a prompt and read output — it maintains a **bidirectional control channel**.

### Messages SDK sends TO the CLI (stdin):

```jsonl
{"type": "user", "session_id": "", "message": {"role": "user", "content": "..."}, "parent_tool_use_id": null}
```

### Control requests (CLI asks SDK for decisions):

```jsonl
{"type": "control_request", "request_id": "req_1_abc", "request": {"subtype": "can_use_tool", ...}}
```

### Control responses (SDK answers CLI):

```jsonl
{"type": "control_response", "response": {"subtype": "success", "request_id": "req_1_abc", "response": {...}}}
```

### Supported control subtypes:

| Subtype | Direction | Purpose |
|---------|-----------|---------|
| `initialize` | SDK to CLI | Send hooks, agents, server info at startup |
| `can_use_tool` | CLI to SDK | Ask permission to use a tool |
| `hook_callback` | CLI to SDK | Invoke hook (PreToolUse, PostToolUse, etc.) |
| `mcp_message` | CLI to SDK | Route JSONRPC to in-process MCP server |
| `interrupt` | SDK to CLI | Interrupt execution |
| `set_permission_mode` | SDK to CLI | Change permission mode mid-session |
| `set_model` | SDK to CLI | Switch model mid-session |
| `rewind_files` | SDK to CLI | Rewind to file checkpoint |
| `mcp_status` | SDK to CLI | Query MCP server status |

---

## 6. All Differences from Direct `claude` Invocation

### Things the SDK adds that direct invocation doesn't have:

| Difference | Impact |
|---|---|
| **`--system-prompt ""`** always sent | Overrides the default system prompt with empty when no prompt specified. Direct `claude` uses its built-in system prompt. |
| **`--setting-sources ""`** by default | Disables user/project/local settings. Direct `claude` loads all settings. |
| **`--output-format stream-json`** | Forces machine-readable output. Direct `claude` defaults to human-readable. |
| **`--input-format stream-json`** | Enables bidirectional JSON protocol on stdin. Direct `claude` reads plain text. |
| **`--verbose`** always set | More detailed logging. |
| **Bidirectional control protocol** | Enables hooks, permission callbacks, in-process MCP servers, mid-session model changes. |
| **Initialize handshake** | SDK sends an `initialize` control request at startup with hooks/agents config. |
| **Bundled CLI preferred** | May use different CLI version than what's on PATH. |
| **Version gate** | Requires CLI >= 2.0.0 (warns but continues if older). |
| **Async write lock** | All stdin writes are serialized through an async lock. |
| **1MB JSON buffer limit** | Messages > 1MB cause `CLIJSONDecodeError`. |

### Things the SDK removes (vs upstream official SDK):

| Removed | Impact |
|---|---|
| **`CLAUDE_CODE_ENTRYPOINT` env var** | CLI doesn't know it's an SDK call, allows Max subscription auth instead of API-key-only |
| **`CLAUDE_AGENT_SDK_VERSION` env var** | No SDK version tracking in CLI telemetry |

### Things the SDK does NOT change:

- No custom signal handlers
- No process group management
- Inherits all parent env vars
- Doesn't modify PATH or NODE_PATH
- Doesn't set `ANTHROPIC_API_KEY` (relies on existing auth)

---

## 7. Complete Flag Map (from `ClaudeAgentOptions` to CLI args)

| SDK Option | CLI Flag | Notes |
|---|---|---|
| `system_prompt` (str) | `--system-prompt <value>` | Empty string if None |
| `system_prompt` (preset+append) | `--append-system-prompt <value>` | |
| `tools` (list) | `--tools "Bash,Read,Edit"` | |
| `tools` (preset) | `--tools "default"` | |
| `allowed_tools` | `--allowedTools "tool1,tool2"` | |
| `disallowed_tools` | `--disallowedTools "tool1,tool2"` | |
| `max_turns` | `--max-turns N` | |
| `max_budget_usd` | `--max-budget-usd N.N` | |
| `model` | `--model <name>` | |
| `fallback_model` | `--fallback-model <name>` | |
| `betas` | `--betas "beta1,beta2"` | |
| `permission_prompt_tool_name` | `--permission-prompt-tool <name>` | Auto-set to "stdio" if `can_use_tool` callback provided |
| `permission_mode` | `--permission-mode <mode>` | default/acceptEdits/plan/bypassPermissions |
| `continue_conversation` | `--continue` | |
| `resume` | `--resume <session_id>` | |
| `settings` + `sandbox` | `--settings <merged_json>` | Sandbox merged into settings JSON |
| `add_dirs` | `--add-dir <path>` (repeated) | |
| `mcp_servers` (dict) | `--mcp-config <json>` | "instance" field stripped for SDK servers |
| `mcp_servers` (path) | `--mcp-config <path>` | |
| `include_partial_messages` | `--include-partial-messages` | |
| `fork_session` | `--fork-session` | |
| `setting_sources` | `--setting-sources "user,project,local"` | Empty string if None |
| `plugins` | `--plugin-dir <path>` (repeated) | |
| `thinking` (adaptive) | `--max-thinking-tokens 32000` | |
| `thinking` (enabled) | `--max-thinking-tokens <budget>` | |
| `thinking` (disabled) | `--max-thinking-tokens 0` | |
| `effort` | `--effort low/medium/high/max` | |
| `output_format` (json_schema) | `--json-schema <json>` | |
| `extra_args` | `--<flag> [value]` | Pass-through for future flags |

---

## 8. Control Protocol Deep Dive

### 8.1 Lifecycle: What Happens on the Wire

The complete sequence from `query("hello")` to final result:

```
SDK (stdin)                              CLI (stdout)
─────────────────────────────────────────────────────────
1. [spawn process with flags]
2. [start background reader task]

3. INITIALIZE REQUEST ───────────────>
   {
     "type": "control_request",
     "request_id": "req_1_a3f2b1c0",
     "request": {
       "subtype": "initialize",
       "hooks": { ... } | null,
       "agents": { ... } | null
     }
   }
                                    <── 4. INITIALIZE RESPONSE
                                        {
                                          "type": "control_response",
                                          "response": {
                                            "subtype": "success",
                                            "request_id": "req_1_a3f2b1c0",
                                            "response": { ... server info ... }
                                          }
                                        }

5. USER MESSAGE ─────────────────────>
   {
     "type": "user",
     "session_id": "",
     "message": {
       "role": "user",
       "content": "hello"
     },
     "parent_tool_use_id": null
   }

6. [close stdin if no hooks/MCP]
   OR
   [keep stdin open for bidirectional]

                                    <── 7. ASSISTANT MESSAGE(s)
                                        {"type": "assistant", "message": {...}}

                                    <── 8. (optional) TOOL PERMISSION REQUEST
                                        {
                                          "type": "control_request",
                                          "request_id": "req_cli_xyz",
                                          "request": {
                                            "subtype": "can_use_tool",
                                            "tool_name": "Bash",
                                            "input": {"command": "ls"},
                                            "permission_suggestions": [...]
                                          }
                                        }

9. PERMISSION RESPONSE ──────────────>
   {
     "type": "control_response",
     "response": {
       "subtype": "success",
       "request_id": "req_cli_xyz",
       "response": {
         "behavior": "allow",
         "updatedInput": {"command": "ls"}
       }
     }
   }

                                    <── 10. MORE MESSAGES...
                                        {"type": "user", "message": {...}}
                                        {"type": "assistant", "message": {...}}

                                    <── 11. RESULT (conversation end)
                                        {
                                          "type": "result",
                                          "subtype": "success",
                                          "duration_ms": 5432,
                                          "total_cost_usd": 0.012,
                                          "session_id": "abc-123",
                                          "num_turns": 3,
                                          "usage": {...}
                                        }

12. [close stdin, wait for exit]
```

### 8.2 Request ID Generation

```python
# query.py:357-358
self._request_counter += 1
request_id = f"req_{self._request_counter}_{os.urandom(4).hex()}"
# Example: "req_1_a3f2b1c0", "req_2_7d9e4f12"
```

Each request gets a monotonically increasing counter + 4 random bytes. The CLI echoes back the `request_id` in its response for correlation.

### 8.3 Message Routing Architecture

All stdout from `claude` flows through a single reader task (`query.py:172-231`):

```
CLI stdout
    |
    v
_read_messages() ──> msg_type == "control_response"? ──> correlate by request_id
    |                                                     set Event, store result
    |
    |──> msg_type == "control_request"? ──> spawn _handle_control_request()
    |                                       in task group (concurrent)
    |
    |──> msg_type == "control_cancel_request"? ──> (TODO: not implemented)
    |
    └──> anything else (user/assistant/system/result/stream_event)
         ──> push to memory channel ──> consumed by receive_messages()
```

Key design points:
- **Control messages are never exposed to the user.** They're intercepted and handled internally.
- **Control requests from CLI are handled concurrently** — each spawned in the task group (`query.py:200-201`), so multiple permission checks can be in-flight.
- **Regular messages go through an `anyio` memory channel** (buffer size 100) — backpressure if consumer is slow.

### 8.4 The `can_use_tool` Callback Protocol

When the CLI wants to use a tool and `--permission-prompt-tool stdio` is set, it sends a control request. The SDK invokes the user's callback:

```python
# query.py:242-283
# CLI sends:
{
    "subtype": "can_use_tool",
    "tool_name": "Bash",
    "input": {"command": "rm -rf /tmp/test"},
    "permission_suggestions": ["allow_once", "allow_always"]
}

# SDK calls user callback:
result = await can_use_tool(
    "Bash",                         # tool name
    {"command": "rm -rf /tmp/test"}, # tool input
    ToolPermissionContext(signal=None, suggestions=["allow_once", "allow_always"])
)

# User returns PermissionResultAllow or PermissionResultDeny:
PermissionResultAllow(updated_input={"command": "ls"})  # can modify input!
PermissionResultDeny(message="Too dangerous", interrupt=True)  # can halt session

# SDK sends back:
{"behavior": "allow", "updatedInput": {"command": "ls"}}
# or
{"behavior": "deny", "message": "Too dangerous", "interrupt": true}
```

The `updatedInput` feature is powerful — the SDK can **rewrite tool inputs** before they execute. This enables input sanitization, auditing, or policy enforcement.

### 8.5 Hook Callback Protocol

Hooks are event-driven callbacks registered at initialization time. The SDK assigns each callback a unique ID:

```python
# query.py:136-140 (during initialize)
callback_id = f"hook_{self.next_callback_id}"  # "hook_0", "hook_1", etc.
self.hook_callbacks[callback_id] = callback     # stored for later dispatch
```

When the CLI triggers a hook:

```python
# CLI sends:
{
    "type": "control_request",
    "request_id": "req_cli_hook_1",
    "request": {
        "subtype": "hook_callback",
        "callback_id": "hook_0",
        "input": { ... hook-specific data ... },
        "tool_use_id": "toolu_abc123"
    }
}

# SDK looks up callback by ID and invokes:
result = await callback(input_data, tool_use_id, {"signal": None})

# User returns HookJSONOutput, e.g.:
{"continue_": True}         # Python field name (keyword conflict)
{"continue_": False, "reason": "blocked by policy"}

# SDK converts Python names to CLI names:
{"continue": True}          # async_ -> async, continue_ -> continue
```

The `_convert_hook_output_for_cli()` function (query.py:34-50) handles the Python to JS keyword mapping.

### 8.6 In-Process MCP Server Bridge

The SDK can host MCP servers **inside the Python process** and bridge them to the CLI via the control protocol. This avoids spawning separate server processes.

**Registration:**
```python
server = create_sdk_mcp_server("my-server", tools=[...])
options = ClaudeAgentOptions(mcp_servers={"my-server": server})
```

**What happens at CLI arg level:**
- The `"instance"` key (holding the Python `McpServer` object) is stripped before passing to `--mcp-config`
- CLI sees `{"type": "sdk"}` with no instance — it knows to route through the control protocol

**Runtime bridging (query.py:391-527):**

```
CLI wants to call MCP tool
    |
    v
CLI sends control_request: {"subtype": "mcp_message", "server_name": "my-server", "message": <JSONRPC>}
    |
    v
SDK._handle_sdk_mcp_request() routes by JSONRPC method:
    |-- "initialize"         -> hardcoded capabilities response
    |-- "tools/list"         -> calls server.request_handlers[ListToolsRequest]
    |-- "tools/call"         -> calls server.request_handlers[CallToolRequest]
    |-- "notifications/initialized" -> acknowledge
    └-- other                -> error -32601
    |
    v
SDK sends control_response with {"mcp_response": <JSONRPC response>}
```

**Known limitation** (noted in code comments): The Python MCP SDK lacks a `Transport` abstraction that TypeScript has. The SDK must manually route each JSONRPC method. New MCP methods (resources, prompts, etc.) require manual code additions.

### 8.7 stdin Lifecycle: When to Close

The SDK has a critical design decision around when to close stdin (query.py:570-602):

- **No hooks, no SDK MCP servers:** Close stdin immediately after sending the user message. The CLI reads the message and runs autonomously.
- **Has hooks or SDK MCP servers:** Keep stdin **open** until the first `result` message arrives. This is because the CLI needs to send control requests back through stdout, and the SDK needs to respond through stdin. Closing stdin prematurely would break the bidirectional protocol.

```python
# query.py:584-600
if self.sdk_mcp_servers or has_hooks:
    # Wait for first result before closing
    with anyio.move_on_after(self._stream_close_timeout):
        await self._first_result_event.wait()
    await self.transport.end_input()
```

The timeout defaults to 60 seconds (from `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT`).

### 8.8 Mid-Session Control Commands

The `ClaudeSDKClient` (interactive mode) exposes control commands that can be sent at any time during a session:

```python
# These all send control_request messages through the open stdin:
await client.interrupt()                          # {"subtype": "interrupt"}
await client.set_permission_mode("bypassPermissions")  # {"subtype": "set_permission_mode", "mode": "..."}
await client.set_model("claude-sonnet-4-5-20250514")    # {"subtype": "set_model", "model": "..."}
await client.rewind_files("msg-uuid-123")         # {"subtype": "rewind_files", "user_message_id": "..."}
status = await client.get_mcp_status()            # {"subtype": "mcp_status"}
```

These are only available in `ClaudeSDKClient` (not the `query()` one-shot function) because they require the stdin to remain open.

### 8.9 Error Propagation in the Control Protocol

**Control request timeout (query.py:374-389):**
```python
with anyio.fail_after(timeout):  # default 60s
    await event.wait()
# TimeoutError -> "Control request timeout: {subtype}"
```

**Control response error (query.py:187-190):**
```python
if response.get("subtype") == "error":
    self.pending_control_results[request_id] = Exception(
        response.get("error", "Unknown error")
    )
```

**Fatal reader error (query.py:220-228):**
When the message reader crashes, it:
1. Signals ALL pending control requests (so they fail fast instead of timing out)
2. Sends an `{"type": "error"}` sentinel through the message channel
3. Sends an `{"type": "end"}` sentinel to terminate iterators

---

## 9. Manual Replication: Talking to `claude` Without the SDK

### 9.1 Minimal One-Shot (Python, no SDK)

This replicates what `query("hello")` does. Uses `asyncio.create_subprocess_exec`
(safe — no shell injection, arguments are passed as a list):

```python
import asyncio, json

async def query_claude(prompt, model=None):
    # Step 1: Build command — these are the hardcoded flags the SDK always sets
    cmd = ["claude",
           "--output-format", "stream-json", "--verbose",
           "--input-format", "stream-json",
           "--system-prompt", "",       # SDK blanks system prompt by default
           "--setting-sources", ""]     # SDK disables settings by default
    if model:
        cmd.extend(["--model", model])

    # Step 2: Spawn process (stdin/stdout piped for bidirectional JSON)
    proc = await asyncio.create_subprocess_exec(
        *cmd, stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    assert proc.stdin and proc.stdout

    # Step 3: Send initialize request (SDK always does this first)
    init_req = {"type": "control_request", "request_id": "req_1_init",
                "request": {"subtype": "initialize", "hooks": None, "agents": None}}
    proc.stdin.write((json.dumps(init_req) + "\n").encode())
    await proc.stdin.drain()

    # Step 4: Read initialize response
    while True:
        line = await proc.stdout.readline()
        if not line: break
        msg = json.loads(line)
        if msg.get("type") == "control_response": break

    # Step 5: Send user message
    user_msg = {"type": "user", "session_id": "",
                "message": {"role": "user", "content": prompt},
                "parent_tool_use_id": None}
    proc.stdin.write((json.dumps(user_msg) + "\n").encode())
    await proc.stdin.drain()

    # Step 6: Close stdin (no hooks/MCP, safe to close immediately)
    proc.stdin.close()

    # Step 7: Read all output messages
    while True:
        line = await proc.stdout.readline()
        if not line: break
        msg = json.loads(line)
        if msg["type"] == "assistant":
            for block in msg.get("message", {}).get("content", []):
                if isinstance(block, dict) and block.get("type") == "text":
                    print(block["text"], end="")
        elif msg["type"] == "result":
            print(f"\n[result] cost=${msg.get('total_cost_usd')}, "
                  f"session={msg.get('session_id')}")
            break
    await proc.wait()

asyncio.run(query_claude("What is 2+2?"))
```

### 9.2 With Permission Handling (can_use_tool equivalent)

Add `--permission-prompt-tool stdio` to the command, and handle `control_request`
messages on stdout by responding on stdin:

```python
# Additional flag needed:
cmd.extend(["--permission-prompt-tool", "stdio"])

# In your read loop, when you see a control_request:
if msg["type"] == "control_request":
    req = msg["request"]
    req_id = msg["request_id"]

    if req["subtype"] == "can_use_tool":
        tool_name = req["tool_name"]
        tool_input = req["input"]

        # Your policy: allow or deny, can also rewrite tool_input
        response = {"type": "control_response", "response": {
            "subtype": "success", "request_id": req_id,
            "response": {"behavior": "allow", "updatedInput": tool_input}
        }}
        # Or to deny:
        # "response": {"behavior": "deny", "message": "Blocked", "interrupt": True}

        proc.stdin.write((json.dumps(response) + "\n").encode())
        await proc.stdin.drain()

# IMPORTANT: Don't close stdin until you see the "result" message!
# The CLI needs stdin open to send permission requests back to you.
```

Key insight: `updatedInput` lets you **rewrite tool arguments** before execution.

### 9.3 Bash One-Liner (Simplest Possible)

```bash
echo '{"type":"user","session_id":"","message":{"role":"user","content":"What is 2+2?"},"parent_tool_use_id":null}' | \
  claude --output-format stream-json --verbose --input-format stream-json \
         --system-prompt "" --setting-sources "" --max-turns 1 2>/dev/null | \
  jq -r 'select(.type == "assistant") | .message.content[] | select(.type == "text") | .text'
```

Note: This skips the initialize handshake. Works for simple one-shots but will fail
if you need hooks or agents.

### 9.4 What You Can Skip vs What You Must Keep

| Element | Required? | What happens if you skip it |
|---|---|---|
| `--output-format stream-json` | **YES** | Output is human text, not parseable JSON |
| `--input-format stream-json` | **YES** | stdin is plain-text prompt, no control protocol |
| `--verbose` | No | Less detail, still works |
| `--system-prompt ""` | No | CLI uses full built-in system prompt (Claude Code default) |
| `--setting-sources ""` | No | CLI loads user/project/local settings |
| Initialize handshake | **YES if hooks/agents** | Without hooks/agents, CLI tolerates skipping it |
| `--permission-prompt-tool stdio` | Only for can_use_tool | CLI uses built-in terminal prompting |

### 9.5 Key Gotchas

1. **JSON buffering**: CLI may split a large JSON object across multiple `readline()` calls.
   The SDK does speculative parsing (try `json.loads()`, if fails, keep buffering).

2. **Don't close stdin too early**: If hooks/MCP registered, CLI needs stdin open for control
   requests. Close only after first `result` message.

3. **Control messages are interleaved**: `control_request` messages arrive mixed with
   `assistant`/`user`/`result`. You must route them, not iterate linearly.

4. **`session_id`**: SDK sends `""`. CLI assigns internally. Real ID in `result` message.

5. **Exit codes**: Non-zero = error. Check `proc.returncode` after `wait()`.

---

## 10. Key Architectural Takeaways

1. **The SDK is a CLI wrapper, not an API client.** It spawns the `claude` binary — it does NOT call the Anthropic API directly.

2. **The bidirectional control protocol is the main value-add.** It enables programmatic permission control, hooks, in-process MCP servers, and mid-session reconfiguration — none of which are possible with a simple `claude -p` invocation.

3. **The fork's main change is removing SDK identity markers** to bypass API-key-only enforcement and allow Max subscription auth.

4. **Settings are suppressed by default** (`--setting-sources ""`), so SDK-driven sessions are isolated from user/project settings unless explicitly opted in.

5. **System prompt is blanked by default** (`--system-prompt ""`), giving full control to the SDK caller rather than inheriting the CLI's built-in system prompt.

6. **The protocol is simple enough to replicate manually** — about 40 lines of Python for a basic one-shot, about 80 lines with permission handling.
