# wrapper-as-launcher

every MCP server WiseHash exposes to an agent runs through a wrapper. the
wrapper is a small bash script. it does three things in order:

1. set environment (PYTHONPATH, language-runtime paths, MCP server config env vars)
2. resolve credentials at exec time (from the operator's sovereign credential store)
3. `exec` the real binary (python, node, or otherwise)

the wrapper does NOT:

- store credentials in plaintext
- log credential values to stdout or stderr
- persist resolved credentials between invocations

the wrapper's three responsibilities are the smallest possible surface that
makes credentials NOT-IN-DOTFILE the default. an agent's `.mcp.json` points at
the wrapper path; the wrapper resolves the credential the moment the MCP server
launches. if the credential rotates, the wrapper picks up the new value on the
next launch — no `.mcp.json` edit required.

## the shape

```bash
#!/usr/bin/env bash
# MCP wrapper: launches <service> MCP server.
# Token resolution: <how — sovereign vault SDK, env, OS keychain, etc.>

REPO_ROOT="$(cd "$(dirname "$0")/.." && pwd)"
SERVICE_DIR="${REPO_ROOT}/services/<service>"
VENV_PYTHON="${SERVICE_DIR}/.venv/bin/python"

# Step 1 — bootstrap environment
export <SERVICE>_URL="<runtime-endpoint>"
export PYTHONPATH="${SERVICE_DIR}"

# Step 2 — resolve credentials
# the credential resolution chain (cache → primary → fallback → error)
# is defined in patterns/credential-resolution-chain.md (v0.2+).
# the wrapper template at templates/wrapper.sh.template demonstrates the
# bootstrap-from-sister-service pattern that avoids repeating
# credential-store connection params in every wrapper.

# Step 3 — exec the real binary
exec "${VENV_PYTHON}" "${SERVICE_DIR}/mcp_server.py"
```

## why bash

bash is universal across the operator hosts the agent runs on. it has no
runtime dependency. it logs nothing by default. it composes with `exec`
cleanly. a wrapper in python or node introduces a runtime dependency for the
wrapper itself, which defeats the purpose of indirection.

## why `exec`

without `exec`, the bash process stays alive holding env vars in a parent
process; with `exec`, the bash process replaces itself with the MCP server
binary. one less process in the tree, no env-var carry-through to anything
spawned later, cleaner signal propagation when the agent shuts the MCP
server down.

## what the wrapper inherits from the agent

nothing by design. the wrapper sets every env var it needs, including the
ones that select tenancy, runtime endpoint, and language paths. inheritance
from the calling agent's environment is a leak surface; explicit env-var
setting is a discipline anchor. the agent runtime may pre-set some env vars
(e.g., a tenant alias for per-tenant credential resolution per the v0.2+
pattern), but those are explicit `WRAPPER_KNOWS_ABOUT` env vars, not
implicit inheritance.

## when this pattern doesn't fit

- **HTTP transport MCP servers.** the wrapper-as-launcher shape assumes the
  MCP server speaks stdio; first-party HTTP MCP servers (Linear, GitHub,
  others) need a stdio bridge wrapper instead. that pattern surfaces in v0.2+
  as `patterns/http-to-stdio-bridge.md`. the wrapper-as-launcher discipline
  still applies to the bridge wrapper — same three steps, same NOT-list.
- **MCP servers shipped as a single binary with no language runtime.** the
  PYTHONPATH / VENV_PYTHON lines are vestigial; the wrapper still needs the
  env-bootstrap + credential-resolution + `exec` shape, just simpler.
- **MCP servers that resolve credentials internally** (rare, usually because
  the server is a thin wrapper over a SDK that already knows how to find
  credentials). the wrapper pattern still applies for the env-bootstrap +
  `exec` discipline; the credential-resolution step becomes a no-op or a
  pre-flight verification only.

## composing with stdio-over-ssh

when the MCP server's authoritative state lives on a different host than the
agent, the local wrapper-as-launcher pattern is wrapped a second time by the
`stdio-over-ssh` pattern (see [`stdio-over-ssh.md`](stdio-over-ssh.md)). the
remote wrapper is even smaller — one `exec ssh` line — and tunnels the local
wrapper's stdio over SSH. the local wrapper-as-launcher discipline is
unchanged; the remote wrapper just adds the network hop.
