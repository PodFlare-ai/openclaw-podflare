# Code sandbox by Podflare — skill for OpenClaw

Run LLM-generated code in a hardware-isolated Podflare Pod (a
dedicated Linux microVM) instead of on your laptop.

This is the OpenClaw / ClawHub [skill package][clawhub] for
[Podflare][pf] — a managed cloud sandbox built for AI agents.
Once installed, any OpenClaw agent (Claude Code, Cursor, Codex,
Cline, or OpenClaw itself) gains a safer alternative to its
default Bash / Terminal tool:

- ~190 ms cold start; ~46 ms hot
- Full Python REPL whose state persists across calls
- `fork(n)` primitive for tree-of-thought in 80 ms
- `egress=False` mode for air-gapped execution on sensitive data
- No filesystem / env / SSH / cloud-CLI access on the host

[clawhub]: https://clawhub.ai
[pf]: https://podflare.ai

## Install

**From ClawHub (recommended):**

```bash
clawhub install podflare
```

**From OpenClaw directly:**

```bash
openclaw skills install podflare
```

**From this repo (if you want to hack on it):**

```bash
git clone https://github.com/PodFlare-ai/openclaw-podflare
openclaw skills install ./openclaw-podflare
```

## Configure

**Step 1 — API key.** Mint a free API key at
[dashboard.podflare.ai/keys](https://dashboard.podflare.ai/keys)
($200 starter credit, no card, 60 seconds). Export it so the
skill can pick it up:

```bash
export PODFLARE_API_KEY=pf_live_...
```

The skill declares `PODFLARE_API_KEY` as its `primaryEnv`, so
OpenClaw will prompt for it on first run if you haven't set it.

**Step 2 — MCP endpoint (one-time, recommended).** So the agent
sees Podflare tools as first-class MCP calls rather than going
through `curl`, add this block to your Claude Code settings at
`~/.claude/settings.json`:

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

Restart the OpenClaw session. The agent now has
`mcp__podflare__run_python`, `mcp__podflare__create_sandbox`,
`mcp__podflare__fork`, etc. alongside the default tools.

If you skip step 2, the skill still works — it falls back to
calling the same MCP endpoint via `curl` from the default Bash
tool. The SKILL.md includes the fallback request shape.

## Verify

Start an OpenClaw session and ask:

> "Run `print(2 + 2)` in a sandbox."

The agent should respond via the `run_python` tool and return
`4`. If it falls back to local Bash instead, check that the
skill's `mcpServers` entry is loaded — `openclaw skills list`
should show `podflare` as active.

## What you get

8 tools exposed by the Podflare MCP server at
`https://mcp.podflare.ai`:

| Tool              | What it does                                     |
| ----------------- | ------------------------------------------------ |
| `create_sandbox`  | Provision a fresh Linux microVM.                 |
| `run_python`      | Execute Python; state persists across calls.     |
| `run_bash`        | Execute Bash; fresh subprocess per call.         |
| `fork`            | Snapshot + spawn N copies of a sandbox (~80 ms). |
| `merge_into`      | Commit a child's state back onto the parent.    |
| `upload`          | Write bytes into the sandbox (base64, ≤6 MB).    |
| `download`        | Read bytes out of the sandbox.                   |
| `destroy_sandbox` | Tear down. Always do this at the end.            |

## Security

Every sandbox is a fresh Podflare Pod — a dedicated microVM with
KVM-backed hardware isolation, its own Linux kernel, and its own
page tables. The Pod has no access to your local filesystem, env
vars, loaded SSH keys, cloud CLI credentials, or browser profile.
A compromised package, a prompt-injected README, or a misbehaving
agent cannot pivot into your environment.

See
[Why Docker isn't enough for LLM-generated code](https://podflare.ai/blog/why-docker-isnt-enough-for-llm-generated-code)
and
[AI coding agent threat model](https://podflare.ai/blog/ai-coding-agent-leaked-my-api-keys)
for the full argument.

## License

MIT. See `LICENSE`.

## Contributing

Issues, PRs, and ClawHub moderation questions all welcome:
<https://github.com/PodFlare-ai/openclaw-podflare>.

For the underlying SDKs and platform:
<https://github.com/PodFlare-ai/podflare>.
