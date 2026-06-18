# credential-resolution chain

a wrapper resolves a credential the moment the MCP server launches (see
[`wrapper-as-launcher.md`](wrapper-as-launcher.md), step 2). *how* it resolves
is a four-link chain, walked in order. every link is explicit, every miss is
logged without exposing a value, and the last link fails loud rather than
starting the server with a missing credential.

## the chain

1. **cache** — is the value already resolved into process env from earlier this
   launch? if so, use it and stop. on a cold launch this link is empty; it
   matters when the same wrapper resolves more than one cipher and a later one
   reuses an earlier connection.
2. **primary store** — call the operator's sovereign secret store (for example a
   self-hosted vault) and fetch the cipher by name. this is the canonical
   source of truth. the store's connection params are themselves bootstrapped,
   not hardcoded in every wrapper (see the sovereign-seed note below).
3. **fallback** — if the primary store is unreachable (network error, store
   down, cipher absent), check a plain env var of the same name. the fallback
   exists for degraded-mode operation and local development. it is not the
   default path and it is never where production credentials live.
4. **error** — if every link misses, raise an error that names the cipher it
   tried to resolve and exit non-zero. the MCP server does not start in a
   partial-credential state.

## the contract

- **resolution happens at exec time, not at config time.** the credential is
  never written into `.mcp.json` or any dotfile; only the cipher *name* is.
  rotating the value in the store needs no config edit — the next launch picks
  up the new value.
- **a miss is logged, a value is never logged.** every link records that it was
  tried and whether it hit, by cipher name only. no link echoes a resolved
  value to stdout or stderr, ever.
- **the error names what to fix, not what was attempted.** "cipher `X` not found
  in store and no fallback env var `X` set" tells the operator exactly what to
  create. it does not print a partial token, a store URL with embedded
  credentials, or a stack trace carrying the value.

## the sovereign-seed bootstrap

the primary store needs its own connection params (store URL, store client
credentials) before it can resolve anything. repeating those in every wrapper
is the same leak problem one layer down. instead, a wrapper sources them from a
single sister service's environment file — one place holds the store
connection, every wrapper borrows it. the cipher values still come from the
store; only the *connection to the store* is seeded.

## why fail loud

a credential that silently falls back to a stale or empty value produces an MCP
server that starts, lists its tools, and fails every call — the worst failure
mode, because the agent papers over it. failing at launch with a named cipher
turns a silent degto-partial-tools into a one-line fix.

## when to deviate

- **MCP servers that resolve credentials internally** (a thin wrapper over an
  SDK that already knows how to find its own credential): links 2–4 collapse
  into a pre-flight check. keep the cache link and the fail-loud contract.
- **fully offline / airgapped hosts** with no primary store: the chain reduces
  to fallback → error. document that the env var *is* the source on that host,
  so no one mistakes it for a leak.

a worked end-to-end run of this chain — including a credential-missing failure —
lives in
[`examples/example-wrapper-with-credential-chain.md`](../examples/example-wrapper-with-credential-chain.md).
