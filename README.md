# mcp-patterns

a pattern library for wiring MCP servers to AI agents in production. wrappers,
templates, and the discipline that keeps credentials, tenancy, and host-locality
from leaking through the cracks.

> this repo wraps work upstream of WiseHash — anthropic's MCP specification,
> the first-party mcp servers (linear, github, and the others), and the
> mcp-remote HTTP-to-stdio bridge. it builds on nate b. jones's framing of
> agent infrastructure as a real, layered stack; alan shurafa's contributions
> to the openclaw / ob1 ecosystem informed several of the credential-tenancy
> patterns here. i am ashton papi; i operate wisehash, and i started codifying
> these wrappers because the agents i run weekly stopped colliding with each
> other once the wiring layer was named.

WiseHash wires MCP servers to agents in five patterns — wrapper-as-launcher,
credential-resolution chain, stdio-over-ssh for cross-host servers, per-tenant
alias-native credentials, and HTTP-to-stdio bridging for first-party servers.
every wrapper is a small bash script that bootstraps environment, resolves
credentials at exec time, and `exec`s the real binary. the patterns exist
because credentials in `.mcp.json` are a credential leak in waiting, and a
plaintext token surface in the most-edited file in the repo is a survival
problem the wiring layer solves once.

what this repo is:
- a set of markdown documents
- a wrapper template
- one sanitized worked example

what this repo is not:
- an mcp server
- an installable package
- a credential store
- a sales methodology

the patterns make their own case through receipts.

## the five patterns

**wrapper-as-launcher** — every MCP server runs through a small bash wrapper.
the wrapper bootstraps environment, resolves credentials at exec time, and
`exec`s the real binary. the agent's `.mcp.json` points at the wrapper path,
not at the binary. this indirection is the load-bearing primitive — see
[`patterns/wrapper-as-launcher.md`](patterns/wrapper-as-launcher.md).

**credential-resolution chain** — the wrapper resolves credentials in a
defined order: cache → primary store → fallback → error. every link in the
chain is explicit, every miss is logged without exposing the value. deferred
to v0.2+; the wrapper template demonstrates the cache → primary shape.

**stdio-over-ssh** — when the MCP server's authoritative state (database,
credential store, runtime config) lives on a different host than the agent,
the local wrapper either fails or spawns a duplicate empty instance. the
remote wrapper variant `exec`s SSH to the canonical host and pipes stdio
both ways — see [`patterns/stdio-over-ssh.md`](patterns/stdio-over-ssh.md).

**per-tenant alias-native credentials** — when the same MCP server needs to
serve multiple agent identities (different bearer tokens, different
audit-trail attribution), the wrapper accepts a tenant alias env var and
resolves the right credential cipher per call. deferred to v0.2+.

**HTTP-to-stdio bridging** — first-party MCP servers (Linear, GitHub) speak
HTTP transport, but the dominant client transport is stdio. `mcp-remote`
bridges the two; the wrapper pattern wraps the bridge with the same
credential-resolution and tenant-alias discipline. deferred to v0.2+.

## the wrapper template

[`templates/wrapper.sh.template`](templates/wrapper.sh.template) is a
directly-runnable bash skeleton with substitution markers. an operator copies
it into their own `scripts/` directory, replaces the `<ANGLE_BRACKET_TOKENS>`
with the service-specific values, and points their agent's `.mcp.json` at the
new wrapper. the template is intentionally bash-only — a Python or Node
template would introduce a runtime dependency for the wrapper itself, which
defeats the indirection point.

## cross-host MCP as a wiring primitive

most MCP server tutorials assume the server and the agent run on the same
machine. real production deployments often don't — credential stores,
databases, or runtime config live on a different host than the agent runtime.
the stdio-over-ssh pattern is the smallest possible bridge across that gap
without duplicating state.

paired with [`wisehash-llc/dispatch-protocol`](https://github.com/wisehash-llc/dispatch-protocol),
the wiring layer becomes a complete substrate: dispatch-protocol routes work
between AI peers; mcp-patterns wires the tools those peers reach for. the two
repos are independent — either is useful on its own — but they compose.

## status

- **version:** v0.1
- **license:** [Apache 2.0](LICENSE) (federation default)
- **provenance:** patterns extracted from several months of production use
  across mixed-model agent peers
- **scope:** stdio transport patterns. HTTP / SSE patterns surface in v0.2+.
- **maintainer:** [ashton-papi](https://github.com/ashton-papi)

WiseHash operates [wisehash.io](https://wisehash.io). this repo is part of
the [`wisehash-llc`](https://github.com/wisehash-llc) federation of focused
public repos covering the operational shape of sovereign agent infrastructure.
