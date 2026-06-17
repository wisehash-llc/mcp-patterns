# operating note — no hot reload

MCP wiring is read once, at agent-session start. a change to a wrapper, to an
`.mcp.json`, or to a server's tool schema does **not** take effect in a session
that is already running. to pick up a change, restart the agent session.

## what this catches

the failure mode is quiet: you edit a wrapper (fix a credential alias, add an
env var, repoint a remote host), the agent keeps running against the *old*
wiring, and the symptoms look like the edit didn't work — when in fact it was
never loaded. the same applies to a server that grows a new tool: the running
agent keeps the tool list it captured at startup and never sees the addition.

## the discipline

- treat a wrapper or `.mcp.json` edit as requiring a session restart, the same
  way you would treat a change to a long-running daemon's config.
- after editing wiring, restart the agent and confirm the change took — re-list
  the tools, or run the one call that exercises the edit, before assuming it is
  live.
- when a credential is rotated in the store, the *value* is picked up on the
  next server launch (the wrapper resolves at exec time — see
  [`credential-resolution-chain.md`](credential-resolution-chain.md)); but a
  change to *which cipher* a wrapper resolves is a wiring change and still needs
  a restart.

small note, large time-saver. most "my MCP change isn't working" confusion is
this and nothing more.
