---
name: code-sandbox-by-podflare
description: Run LLM-generated code in a hardware-isolated Podflare Pod microVM instead of on the host machine. Gives the agent a persistent Python REPL, ~80 ms fork(n) for tree-of-thought branching, and an optional no-egress mode for air-gapped execution over sensitive data. Drop-in safer replacement for the default Bash / Terminal tool.
version: 1.0.0
homepage: https://podflare.ai
metadata:
  openclaw:
    requires:
      env:
        - PODFLARE_API_KEY
      bins:
        - curl
    primaryEnv: PODFLARE_API_KEY
    emoji: "🔥"
    os:
      - darwin
      - linux
      - win32
---

# Code sandbox by Podflare

This skill teaches the agent to offload code execution to a
managed Podflare Pod — a hardware-isolated microVM at the other
end of an MCP server at `https://mcp.podflare.ai`.

- **~190 ms cold start**, ~46 ms hot (on an already-open sandbox)
- Real Linux: `pip install`, `apt`, full filesystem
- Python REPL whose state persists across calls
- `fork(n)` primitive for tree-of-thought
- No access to the host's filesystem / env vars / SSH keys / cloud CLI

## Prerequisite (one-time, ~30 seconds)

The skill can reach Podflare two ways. Pick one and configure it
once:

### Option A — via Claude Code's MCP config (recommended)

Add the Podflare MCP server to your Claude Code settings so the
agent gets `mcp__podflare__run_python`,
`mcp__podflare__create_sandbox`, etc. as first-class tools.

Edit `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "podflare": {
      "url": "https://mcp.podflare.ai",
      "headers": {
        "Authorization": "Bearer ${PODFLARE_API_KEY}"
      }
    }
  }
}
```

Restart your OpenClaw session. The `mcp__podflare__*` tools will
appear alongside the default ones.

### Option B — via curl from the default Bash tool (fallback)

If you can't edit the Claude Code config, the agent can talk to
`mcp.podflare.ai` directly over JSON-RPC using the existing Bash
tool and `curl`. See the "curl fallback" section below for the
exact request shape.

## When to use this skill

Prefer Podflare over the default Bash / Terminal tool for:

- Running code the model just generated, that the user has not
  hand-reviewed.
- `pip install`, `npm install`, `apt` — any package manager that
  can run arbitrary post-install hooks from a compromised package.
- Parsing or transforming untrusted input (URLs, files, API
  responses).
- Running tests or linters over LLM-authored code.
- Any "let me just try this and see what happens" exploration.

The sandbox has no route to the host's filesystem, env vars,
cloud CLI, or SSH agent. The worst the generated code can do is
misbehave inside a disposable VM.

## When the default Bash is still appropriate

Keep Bash for:

- Commands the user explicitly asked to run on their machine
  (`git status`, `make test`, editor operations).
- File edits in the user's workspace (the native `Read` /
  `Write` / `Edit` tools are scoped to the workspace; use those).
- Operations that need host resources (local DB, docker daemon,
  specific file descriptors).

Rule of thumb: **Podflare for untrusted or LLM-generated code,
host Bash for trusted host-local operations the user asked for.**

## Tools exposed (via Option A — MCP)

Once the MCP server is configured, the agent has these tools:

- **`mcp__podflare__create_sandbox`** — provision a fresh VM.
  `template: "default"` or `"python-datasci"` (pandas / numpy /
  scipy / matplotlib preloaded). Returns `sandbox_id`.
- **`mcp__podflare__run_python`** — run Python. State persists
  across calls.
- **`mcp__podflare__run_bash`** — run Bash. Fresh subprocess per
  call; no shell-variable persistence.
- **`mcp__podflare__fork`** — snapshot + spawn N children in
  ~80 ms. Each child inherits the parent's full REPL state.
- **`mcp__podflare__merge_into`** — commit a child's state back
  onto the parent.
- **`mcp__podflare__upload`** / **`download`** — move bytes in
  and out (base64, ≤6 MB raw).
- **`mcp__podflare__destroy_sandbox`** — tear down. Always do this
  at the end.

## Lifecycle pattern (via MCP)

```
create_sandbox()      → sandbox_id
run_python(...)       → stdout       (state kept)
run_python(...)       → stdout       (globals still alive)
(optional) fork(n=5)  → children
... branch exploration ...
merge_into(parent, winner)
destroy_sandbox(each loser)
destroy_sandbox(parent)  # when task is fully done
```

## curl fallback (Option B)

If MCP is not configured, the agent can reach the same endpoint
with Bash + curl. The JSON-RPC 2.0 shape is:

```bash
# Create a sandbox
curl -sX POST https://mcp.podflare.ai \
  -H "Authorization: Bearer $PODFLARE_API_KEY" \
  -H "content-type: application/json" \
  -d '{
        "jsonrpc":"2.0","id":1,"method":"tools/call",
        "params":{"name":"create_sandbox","arguments":{"template":"default"}}
      }'

# Run Python in it
curl -sX POST https://mcp.podflare.ai \
  -H "Authorization: Bearer $PODFLARE_API_KEY" \
  -H "content-type: application/json" \
  -d '{
        "jsonrpc":"2.0","id":2,"method":"tools/call",
        "params":{"name":"run_python","arguments":{
          "sandbox_id":"SANDBOX_ID_FROM_ABOVE",
          "code":"print(2+2)"
        }}
      }'

# Destroy when done
curl -sX POST https://mcp.podflare.ai \
  -H "Authorization: Bearer $PODFLARE_API_KEY" \
  -H "content-type: application/json" \
  -d '{
        "jsonrpc":"2.0","id":9,"method":"tools/call",
        "params":{"name":"destroy_sandbox","arguments":{"sandbox_id":"…"}}
      }'
```

The agent should prefer Option A (MCP) when available — it gives
cleaner tool-call traces and better error handling. Use the curl
fallback only as a last resort when MCP is not configured.

## Passing credentials into the sandbox

If the generated code legitimately needs an API key (e.g. the user
asked the agent to call OpenAI from inside the sandbox), the user
types it into the chat explicitly and the agent inlines it:

```python
import os, openai
os.environ["OPENAI_API_KEY"] = "sk-user-provided-scoped-key"
client = openai.OpenAI()
```

**The sandbox cannot reach host env vars.** Do not assume
`os.environ["OPENAI_API_KEY"]` is set inside the VM — it isn't,
unless you put it there. This is a feature: production
credentials never cross into LLM-generated code execution.

## Air-gapped mode (optional)

For workloads over sensitive data — PII, PHI, financial, academic
restricted-use — pass `egress=False` in the `create_sandbox`
arguments. The sandbox runs normally but has no outbound network.
See `https://podflare.ai/blog/air-gapped-sandbox-sensitive-data-analysis`
for the full playbook.

## Examples

### Basic usage

```
user: "Compute the standard deviation of [1,2,3,4,5,6,7,8,9,10]."

agent (via MCP):
  mcp__podflare__create_sandbox(template="default") → sandbox_id="sb_abc"
  mcp__podflare__run_python(sandbox_id="sb_abc", code=
    "import statistics; print(statistics.stdev([1,2,3,4,5,6,7,8,9,10]))")
    → stdout: "3.0276503540974917"
  mcp__podflare__destroy_sandbox(sandbox_id="sb_abc")
```

### Tree-of-thought with fork

```
create_sandbox()                → "sb_parent"
run_python(load expensive df)   # paid once
fork(sandbox_id="sb_parent", n=3) → ["sb_c1","sb_c2","sb_c3"]
(parallel run on each)
merge_into("sb_parent", winner)
destroy_sandbox(loser_a); destroy_sandbox(loser_b); destroy_sandbox("sb_parent")
```

## Further reading

- Podflare docs: https://docs.podflare.ai
- OpenClaw integration guide: https://docs.podflare.ai/integrations/openclaw
- Why Docker isn't enough for LLM-generated code:
  https://podflare.ai/blog/why-docker-isnt-enough-for-llm-generated-code
- AI coding agent threat model (API-key leaks, prompt injection):
  https://podflare.ai/blog/ai-coding-agent-leaked-my-api-keys
- Head-to-head benchmark vs E2B, Daytona, Blaxel:
  https://podflare.ai/blog/cloud-sandbox-benchmark-e2b-daytona-podflare

## Support

Issues: https://github.com/PodFlare-ai/openclaw-podflare/issues
Email: hello@podflare.ai
Free API key ($200 starter credit):
https://dashboard.podflare.ai/keys
