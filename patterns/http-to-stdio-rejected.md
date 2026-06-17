# http-to-stdio bridging — evaluated and rejected

this is not a pattern. it is a receipt for a pattern we evaluated and chose not
to adopt, kept here so the absence is a decision, not an oversight.

## what it would have been

first-party MCP servers (the hosted ones — issue trackers, code forges, and the
like) speak HTTP transport. a stretch of MCP tutorials wrap them in a local
stdio bridge — an `mcp-remote` / `mcp-proxy` style process that accepts stdio
from the agent and re-emits it as HTTP to the remote server — so that every MCP
the agent talks to looks like a uniform stdio wrapper.

on the surface it composes neatly with the rest of this repo: same
wrapper-as-launcher shape, same credential-resolution and alias discipline,
wrapped around the bridge instead of a local binary.

## why we rejected it

the dominant first-party agent CLIs already speak HTTP transport natively. when
the client handles HTTP directly, the bridge buys nothing and costs three
things:

- **a runtime dependency** — the bridge is an npm package, which reintroduces
  exactly the kind of runtime dependency the bash-only wrapper exists to avoid.
- **an extra process in the tree** — one more thing to launch, supervise, and
  reap; one more place for a hung connection or an orphaned process to hide.
- **a second credential surface** — the bridge sits in the auth path, so the
  token now transits an extra process layer instead of going client-to-server.

none of those costs is offset by a sovereignty or reliability gain. the bridge
solves a uniformity problem (everything-looks-like-stdio) that is cosmetic, not
operational.

## what we do instead

let the client speak HTTP natively. point the agent's config at the remote MCP's
URL directly; carry the credential through the config's own per-server env, not
through a bridge process. the wrapper-as-launcher pattern stays canonical for
**stdio-transport servers only** — that is its scope, stated plainly, rather
than stretched to cover a transport it was never the right tool for.

## when this receipt could flip

if a future client lost native HTTP transport, or if a deployment needed an
auditable single choke point in front of several remote MCPs, the bridge's costs
might start earning their keep. they do not today. revisit the receipt if the
client landscape changes; do not adopt the bridge by default.
