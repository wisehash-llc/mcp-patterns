# example: wrapper-with-credential-chain

a fully-sanitized worked example of the wrapper-as-launcher pattern applied to
a fictional `example-service` MCP server. the example shows the wrapper
shape, the credential-resolution chain in action, and the failure-mode
discipline when a credential is missing.

> this example is authored from scratch as a teaching artifact. it is NOT
> lifted from any specific production wrapper. the service name, paths, and
> credential names are fictional. the operational shape — env-bootstrap,
> credential-resolution, `exec` — matches the canonical pattern from
> [`patterns/wrapper-as-launcher.md`](../patterns/wrapper-as-launcher.md)
> exactly.

## scenario

an operator runs an `example-service` MCP server that exposes one tool —
`example_query` — to AI agents. the MCP server is a Python application
living at `services/example-service/mcp_server.py` in the operator's
monorepo. authentication uses a bearer token from the operator's sovereign
credential vault.

the operator wants:

1. the wrapper to launch the MCP server with the right Python venv
2. the bearer token resolved at exec time (not stored in `.mcp.json`)
3. multiple agents to share the same MCP server but with different bearer
   tokens (one bearer per agent identity, for audit-trail attribution)
4. graceful failure if the credential vault is down or the bearer is missing

## the wrapper

`scripts/mcp-example-service.sh`:

```bash
#!/usr/bin/env bash
# MCP wrapper: launches example-service MCP server.
# Token resolution: sovereign vault SDK, alias-resolved per-agent.
# Caller may pre-set EXAMPLE_SERVICE_TENANT_ALIAS to select a tenant cipher.

set -euo pipefail

REPO_ROOT="$(cd "$(dirname "$0")/.." && pwd)"
SERVICE_DIR="${REPO_ROOT}/services/example-service"
VENV_PYTHON="${SERVICE_DIR}/.venv/bin/python"

# Step 1: bootstrap environment
export EXAMPLE_SERVICE_URL="${EXAMPLE_SERVICE_URL:-http://localhost:9100}"
export PYTHONPATH="${SERVICE_DIR}"

# Tenant alias — caller selects the bearer cipher; defaults to a per-machine
# default cipher that maps to a low-privilege read-only token.
export EXAMPLE_SERVICE_TENANT_ALIAS="${EXAMPLE_SERVICE_TENANT_ALIAS:-EXAMPLE_TOKEN_DEFAULT}"

# Step 2: resolve credentials
# bootstrap credential-store connection from a sister service's .env
# (the sovereign-seed pattern) — the sister service has the vault URL +
# vault client credentials needed by the SDK below.
BOOTSTRAP_ENV="${REPO_ROOT}/services/intake-service/.env"
if [ -f "${BOOTSTRAP_ENV}" ]; then
    set -a
    # shellcheck disable=SC1090
    source "${BOOTSTRAP_ENV}"
    set +a
fi

# Step 3: exec the real binary
exec "${VENV_PYTHON}" "${SERVICE_DIR}/mcp_server.py"
```

## the credential-resolution chain

the wrapper itself does not resolve the bearer — that happens inside
`mcp_server.py` when the SDK is initialized. the chain is:

1. **process env cache**: SDK checks if `EXAMPLE_TOKEN` is already in process
   env (no). first run, no cache.
2. **alias lookup**: SDK reads `EXAMPLE_SERVICE_TENANT_ALIAS` (set by the
   wrapper to `EXAMPLE_TOKEN_DEFAULT` unless the caller overrode). resolves
   the alias to the cipher name.
3. **vault SDK call**: SDK calls the credential vault's REST API with the
   client credentials sourced from `intake-service/.env` (the sovereign-seed
   bootstrap), authenticates, and fetches the cipher's value.
4. **fallback to env**: if the vault call fails (network error, vault down,
   cipher not found), SDK falls back to checking `EXAMPLE_TOKEN` directly in
   process env. on this first run, that env var is unset.
5. **error**: if all four steps fail, the MCP server raises a clear error
   message naming the cipher it tried to resolve. the agent receives a
   tool-list-unavailable response and surfaces the failure to the operator.

the chain order is explicit, every miss is logged without exposing the
value, and the error message names what to fix without leaking what was
attempted.

## per-agent invocation

an operator running two agents — one for development tooling (low-privilege
bearer), one for production query (high-privilege bearer with audit trail) —
configures the agents' `.mcp.json` files differently:

**agent 1 (development tooling)**, `~/.config/agent-dev/mcp.json`:

```json
{
  "mcpServers": {
    "example-service": {
      "command": "/Users/operator/repo/scripts/mcp-example-service.sh",
      "env": {
        "EXAMPLE_SERVICE_TENANT_ALIAS": "EXAMPLE_TOKEN_DEV"
      }
    }
  }
}
```

**agent 2 (production query)**, `~/.config/agent-prod/mcp.json`:

```json
{
  "mcpServers": {
    "example-service": {
      "command": "/Users/operator/repo/scripts/mcp-example-service.sh",
      "env": {
        "EXAMPLE_SERVICE_TENANT_ALIAS": "EXAMPLE_TOKEN_PROD"
      }
    }
  }
}
```

both agents share the same wrapper script. the `EXAMPLE_SERVICE_TENANT_ALIAS`
env var (per-agent in `.mcp.json`) selects which cipher the wrapper resolves.
no bearer values appear in either `.mcp.json` — only cipher *names*.

if the operator rotates `EXAMPLE_TOKEN_PROD` in the vault, no `.mcp.json`
edit is required. the next launch of agent 2's MCP server picks up the new
value automatically.

## halt-and-report discipline (a credential-missing example)

suppose the operator forgets to create `EXAMPLE_TOKEN_PROD` in the vault
before agent 2 starts. on first invocation:

1. wrapper launches: env bootstrap clean, sister-service `.env` loaded,
   `exec` python.
2. MCP server starts, SDK initializes, calls vault for `EXAMPLE_TOKEN_PROD`.
3. vault returns 404 (cipher not found).
4. SDK falls back to env — `EXAMPLE_TOKEN` not set.
5. MCP server raises `CredentialNotFound: cipher 'EXAMPLE_TOKEN_PROD' not
   found in vault and no fallback env var EXAMPLE_TOKEN set`.
6. MCP server exits with non-zero status. agent receives connection-closed
   from the wrapper subprocess.
7. agent logs the wrapper exit and surfaces a tool-list-unavailable response
   to the operator.

the failure is loud, the error message names exactly what to fix, and no
credential value is ever logged or echoed to stdout/stderr. the operator
creates the cipher in the vault, restarts the agent, and the next launch
succeeds — no `.mcp.json` edit, no wrapper edit.

this is the discipline anchor: **a credential failure surfaces immediately
with a clear remediation, never silently degrades to a partial-tools state
that an agent might paper over.**

## what this example does NOT show

- **stdio-over-ssh**: this example assumes the MCP server runs locally. when
  the example-service database lives on a different host than the agent, the
  remote wrapper variant from [`stdio-over-ssh`](../patterns/stdio-over-ssh.md)
  wraps this wrapper.
- **HTTP transport servers**: this example uses a stdio-native MCP server.
  first-party HTTP MCP servers (Linear, GitHub) are handled by the client's
  native HTTP transport; bridging them to stdio was evaluated and rejected
  ([`http-to-stdio-rejected.md`](../patterns/http-to-stdio-rejected.md)).
- **deeper credential-chain semantics**: cache invalidation, refresh-token
  rotation, store-backend-specific quirks. the chain itself is documented in
  [`credential-resolution-chain.md`](../patterns/credential-resolution-chain.md).

the example is intentionally narrow — one wrapper, one MCP server, one
credential, two agents. the patterns compose without losing the discipline.
