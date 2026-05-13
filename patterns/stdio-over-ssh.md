# stdio-over-ssh

MCP servers usually run on the same machine as the agent. but real production
deployments often have the MCP server's authoritative state — database,
credential store, runtime config — on a different host than the agent. this
pattern bridges that gap without duplicating state.

## the problem

an agent on workstation A wants to call an MCP server whose database lives on
server B. the standard wrapper at `scripts/mcp-<name>.sh` assumes local state
(local DB path, local credential store). running it on workstation A either:

- fails (no local DB, no local credentials)
- spawns a duplicate empty MCP server instance against an empty local DB

both are wrong. the agent needs a single canonical view of the state, which
lives on server B.

## the pattern

create a `<name>-remote.sh` wrapper alongside the local one. the remote wrapper
contains exactly one line of work:

```bash
#!/usr/bin/env bash
# MCP wrapper (remote): tunnels stdio over SSH to <server-B>.
# Use this on machines that don't host the MCP server's local state.
# On <server-B> itself, use scripts/mcp-<name>.sh directly.

exec ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 <server-B> \
  /path/to/scripts/mcp-<name>.sh
```

the agent's `.mcp.json` on workstation A points at `mcp-<name>-remote.sh`. SSH
pipes stdio both ways — the MCP server runs on server B with full state access,
the agent on workstation A sees it as a normal stdio MCP.

## prerequisites

- passwordless SSH from workstation A to server B (key-based, agent-resident
  or keychain-resident)
- the local wrapper (`scripts/mcp-<name>.sh`) exists and is functional on
  server B
- server B is reachable on the network from workstation A

## caveats

- if server B is down, the MCP server fails — there is no local fallback by
  default. for some MCP servers (those with local degraded-mode capability),
  reverting `.mcp.json` to point at a local empty wrapper is a degraded
  fallback. for credential-bound MCP servers (like a sovereign vault), there
  is no degraded fallback by design.
- SSH connection multiplexing (`ControlMaster` in `~/.ssh/config`) reduces
  reconnect overhead if the wrapper fires frequently. consider it for any
  wrapper that handles >5 invocations per minute.
- the SSH `ServerAliveInterval` settings keep the tunnel alive for long-running
  MCP sessions (the default is no keepalive, which causes connection drops on
  network blips).
- the remote wrapper inherits the local-side wrapper-as-launcher discipline
  by transitivity — the local-side wrapper still resolves credentials at exec
  time on server B, still avoids plaintext leaks, still `exec`s the real
  binary.

## when NOT to use

- when the MCP server's state is supposed to be host-local (single-user
  isolation, sandbox, airgap-intent host)
- when the cross-host call would violate a tier boundary (e.g., a sandboxed
  agent on a non-Internet host should not tunnel out)
- when the latency cost of the SSH hop is meaningful (typical SSH overhead is
  ~5-50ms per stdio round-trip, fine for most MCP workloads but punishing for
  high-frequency tool calls)

the pattern is a wiring primitive, not a default. apply it when state-locality
demands it; do not apply it when host-isolation is the design.

## variants

- **direct user account.** the simplest shape — `exec ssh user@host
  /path/to/wrapper.sh`. assumes the user has the wrapper in PATH or at a known
  absolute path.
- **per-agent SSH user.** when audit-trail attribution matters (multi-agent
  workspace, regulated environment), each agent gets its own SSH user on the
  remote host. the remote wrapper carries the agent's user identity through
  to the credential-resolution layer.
- **bastion-hop.** when server B is behind a bastion, the remote wrapper
  becomes `exec ssh -J bastion@gateway server-B /path/to/wrapper.sh`. SSH
  ProxyJump handles the hop transparently.
- **non-SSH transport.** for environments where SSH is not the canonical
  cross-host tunnel (e.g., HTTPS-only with mTLS, or a service-mesh sidecar),
  the same pattern applies with the transport substituted. the principle —
  bridge stdio across the network without duplicating state — is independent
  of SSH specifically.

## verification

after wiring the remote wrapper:

1. on workstation A, point `.mcp.json` at `mcp-<name>-remote.sh`.
2. start the agent. the agent should see the MCP server respond to its tool
   list call within ~1-2 seconds (SSH connection setup + MCP server start).
3. run a tool call that touches authoritative state (e.g., a query that
   returns rows from the canonical database). verify the result matches what
   `ssh server-B /path/to/wrapper.sh` would have returned directly.
4. kill the agent. verify the SSH connection cleanly closes (the `exec` in
   the remote wrapper means the SSH process is the wrapper process — when
   the agent closes stdio, SSH closes the tunnel, which terminates the MCP
   server on server B).

if step 4 leaves orphan SSH processes on workstation A or orphan MCP server
processes on server B, the wrapper is missing the `exec` keyword somewhere
or has a subprocess fork that breaks the signal-propagation chain.
