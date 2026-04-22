---
name: podflare
description: >
  Run any LLM-generated code in a hardware-isolated Firecracker
  microVM instead of on the host machine. Hands the agent a
  persistent Python REPL, an 80 ms fork(n) for tree-of-thought,
  and an egress=False mode for air-gapped execution over
  sensitive data. Drop-in safer replacement for the default Bash
  / Terminal tool in Claude-Code-backed agents.
homepage: https://podflare.ai
user-invocable: true
metadata:
  openclaw:
    requires:
      env: [PODFLARE_API_KEY]
    primaryEnv: PODFLARE_API_KEY
    emoji: "🔥"
    os: [darwin, linux, win32]
    mcpServers:
      podflare:
        url: https://mcp.podflare.ai
        headers:
          Authorization: "Bearer ${PODFLARE_API_KEY}"
---

# Podflare sandbox

This skill gives the agent a managed Podflare sandbox via the MCP
server at `https://mcp.podflare.ai`. Each sandbox is a fresh,
hardware-isolated Firecracker microVM — real kernel, real
filesystem, ~190 ms round-trip, `pip install` anything.

## When to reach for this skill

Prefer `run_python` (this skill) over the default `Bash` / shell
tool for anything the user has not hand-reviewed, and in
particular for:

- Executing code the model just generated
- Running `pip install`, `npm install`, `apt`, any package manager
- Trying out a new library or API
- Parsing / transforming untrusted input (URLs, files, responses)
- Running tests or linters against LLM-authored code
- Any kind of "let me just try this and see what happens"

The sandbox has **no access** to the host's filesystem, env vars,
cloud CLI credentials, or SSH keys. The worst the generated code
can do is misbehave inside a disposable VM that is getting
destroyed anyway.

## When the default Bash is still appropriate

Do not route through this skill when:

- The user explicitly wants to run a command on their own machine
  (e.g. `git status`, `make test` in their repo, editor commands)
- You're manipulating files in the user's workspace (use the
  native `Read` / `Write` / `Edit` tools; those are scoped to
  the workspace, not to arbitrary code execution)
- The command genuinely needs host resources (local database,
  specific file descriptors, docker daemon, etc.)

The pattern is: **sandbox for untrusted or LLM-generated code,
host for trusted host-local operations the user asked for.**

## Tools this skill exposes

Once installed, the agent gains these tools (all via the Podflare
MCP server):

- **`create_sandbox`** — provision a new VM. `template: "default"`
  or `"python-datasci"` (pandas/numpy/scipy/matplotlib preloaded).
  Returns `sandbox_id`.
- **`run_python`** — execute Python in the sandbox.
  **State persists** across calls: variables, imports, open files.
  Returns `stdout`, `stderr`, `exit_code`.
- **`run_bash`** — execute a Bash snippet. Shell state does **not**
  persist across calls (fresh subprocess each time).
- **`fork`** — snapshot a sandbox and spawn N copies. Each child
  starts with the parent's full Python REPL and filesystem state.
  ~80 ms server-side for n=5.
- **`merge_into`** — commit a child's state back onto the parent.
  Parent id keeps working, child is destroyed.
- **`upload`** / **`download`** — move bytes in and out (base64
  encoded, up to ~6 MB raw per call).
- **`destroy_sandbox`** — tear down. Always do this at the end.

## Lifecycle pattern

```
create_sandbox()        →  sandbox_id
run_python(...)         →  stdout
run_python(...)         →  stdout  (state persists!)
(optional) fork(n=5)    →  children, each with parent state
... branch exploration ...
merge_into(parent, winner)
destroy_sandbox(loser_a); destroy_sandbox(loser_b); ...
destroy_sandbox(parent)  # when task is done
```

Destroying at the end is the only hygiene rule — billed compute
stops the moment the VM is destroyed.

## Passing credentials into the sandbox

If your generated code legitimately needs an API key (e.g. the
user asked you to call OpenAI from inside the sandbox), request
it explicitly from the user and pass it via the Python code
itself:

```python
import os, openai
os.environ["OPENAI_API_KEY"] = "sk-user-provided-scoped-key"
client = openai.OpenAI()
...
```

**Do not** access host env vars. The sandbox cannot reach them;
if you need `OPENAI_API_KEY`, the user should type it into the
chat explicitly or use a dedicated, scoped key for agent use.

## Air-gapped mode

For workloads involving sensitive data (PII, PHI, financial
records, proprietary datasets), create the sandbox with
`egress=False` in the body of your `create_sandbox` call, which
detaches the VM from the public internet. The sandbox still has
full compute, full filesystem, full REPL — but no outbound
network. See `/blog/air-gapped-sandbox-sensitive-data-analysis`
on podflare.ai for the threat model and use cases.

## Examples

### Basic usage

```
user: "Compute the standard deviation of [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]."

agent: create_sandbox(template="default")    → sandbox_id="sb_abc"
agent: run_python(sandbox_id="sb_abc", code=(
  "import statistics; "
  "print(statistics.stdev([1,2,3,4,5,6,7,8,9,10]))"
))
  → stdout: "3.0276503540974917"
agent: destroy_sandbox(sandbox_id="sb_abc")
```

### Tree-of-thought with fork

```
user: "Try 3 different approaches to solve <problem> and tell me which works."

agent: create_sandbox() → "sb_parent"
agent: run_python(...)  # load expensive context once
agent: fork(sandbox_id="sb_parent", n=3) → ["sb_c1","sb_c2","sb_c3"]
(parallel)
  run_python(sb_c1, approach_1)
  run_python(sb_c2, approach_2)
  run_python(sb_c3, approach_3)
agent: # pick winner, report back to user
agent: destroy_sandbox(sb_c1); destroy_sandbox(sb_c2); destroy_sandbox(sb_c3)
agent: destroy_sandbox(sb_parent)
```

### Data analysis with persistent state

```
user: "Load sales.parquet and answer some questions."

agent: create_sandbox(template="python-datasci") → "sb_xyz"
agent: upload(sb_xyz, "/data/sales.parquet", <base64>)
agent: run_python(sb_xyz, "import pandas as pd; df = pd.read_parquet('/data/sales.parquet')")

# Later turns — df is still in memory, no re-load:
agent: run_python(sb_xyz, "print(df.groupby('region').revenue.sum())")
agent: run_python(sb_xyz, "print(df.describe())")
```

## Further reading

- Podflare docs: https://docs.podflare.ai
- Why Docker isn't enough for LLM-generated code:
  https://podflare.ai/blog/why-docker-isnt-enough-for-llm-generated-code
- AI coding agent threat model (leaked API keys, prompt
  injection, etc.): https://podflare.ai/blog/ai-coding-agent-leaked-my-api-keys
- Cloud sandbox benchmark (vs E2B, Daytona, Blaxel):
  https://podflare.ai/blog/cloud-sandbox-benchmark-e2b-daytona-podflare

## Support

Issues: https://github.com/PodFlare-ai/openclaw-podflare/issues
Email: hello@podflare.ai
