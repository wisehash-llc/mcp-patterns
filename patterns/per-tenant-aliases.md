# per-tenant alias-native credentials

one MCP server binary, many agent identities. when several agents share a
single MCP server but must authenticate as *different* principals — different
bearer tokens, different audit-trail attribution — the wrapper resolves a
different credential cipher per launch, selected by an alias env var. the agent
never holds a token; it holds the *name of an alias*.

## the problem

two agents — call them `agent-a` and `agent-b` — both talk to the same MCP
server. you want every action `agent-b` takes to be attributable to `agent-b`
in the server's audit trail, distinct from `agent-a`. the naive fixes both
fail:

- **one shared token** — actions are indistinguishable; the audit trail is
  useless and revoking one agent revokes both.
- **a second copy of the server per agent** — duplicates state and doubles the
  operational surface for an identity problem, not a state problem.

## the pattern

the wrapper reads an alias env var and uses it to select which cipher to
resolve from the credential store:

```bash
# caller (the agent's .mcp.json) sets SERVICE_TOKEN_ALIAS to pick an identity.
# default is a low-privilege cipher, not a privileged one.
export SERVICE_TOKEN_ALIAS="${SERVICE_TOKEN_ALIAS:-SERVICE_TOKEN_DEFAULT}"
```

the alias names a cipher like `SERVICE_TOKEN_AGENT_A` or `SERVICE_TOKEN_AGENT_B`;
the credential-resolution chain ([`credential-resolution-chain.md`](credential-resolution-chain.md))
fetches that cipher's value at exec time. each agent's `.mcp.json` sets a
different alias:

```json
{
  "mcpServers": {
    "service": {
      "command": "/path/to/scripts/mcp-service.sh",
      "env": { "SERVICE_TOKEN_ALIAS": "SERVICE_TOKEN_AGENT_B" }
    }
  }
}
```

both agents point at the same wrapper. only the alias differs. no token value
appears in either config — only cipher *names*.

## audit attribution closes the loop

selecting the right token is half the pattern; the server completing the loop is
the other half. the server resolves the *incoming* token back to an identity and
attributes every operation to it:

```
incoming bearer token  →  lookup in {token: identity} map  →  identity
                                                              ↓
                          every write tagged with that identity
```

the map lives server-side, keyed by token, valued by identity. a request
arriving with `agent-b`'s token is recorded as `agent-b`. revoking `agent-b`
means rotating one cipher; `agent-a` is untouched.

## the default must be least-privilege

the alias defaults to a low-privilege, ideally read-only cipher. an agent
launched without an explicit alias gets the safe identity, not a privileged
one — a missing env var should never fail *open*.

## partial adoption is fine

not every server needs this. a single-agent server hardcodes one cipher name
and skips the alias indirection entirely; retrofitting the alias is a one-line
change when a second identity appears. adopt the pattern when a second
principal shows up, not preemptively.

## sanitization note

the cipher names, agent identities, and token-to-identity map in this doc are
illustrative placeholders. in a real deployment these are operator-specific and
never committed to a public artifact — only the *shape* travels, not the names.
