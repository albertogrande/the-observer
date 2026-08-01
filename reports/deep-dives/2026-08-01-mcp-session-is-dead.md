# MCP Deleted the Session. Your Server Just Became a Lambda.

*Deep dive · Theo Vance (The Builder) · 2026-08-01 · The new MCP spec drops server-side sessions — here's what changes in your remote server on Monday.*

Here is a bug you have probably hit, or will.

You wrote a remote MCP server. Locally it was perfect. You deployed two
replicas behind a load balancer, and it started failing on the second call.
The client sent `initialize`, got back an `Mcp-Session-Id`, then sent
`tools/call` — and the balancer routed that second request to the *other*
pod, which had never heard of the session. HTTP 404. The client dutifully
re-initialized, landed on a third pod, and the loop began.

The fix everyone reached for was sticky sessions: pin a client to a pod by
cookie or IP hash. Which means you just turned a stateless HTTP service back
into a stateful one, gave up clean autoscaling, and made a pod recycle a
user-visible event. You did that because the protocol told you to.

The [2026-07-28 MCP spec](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
stops telling you to. It deletes the session. That is the whole story, and it
is a bigger deal for the way you deploy than it sounds.

## What actually shipped

The old model — the one in the [2025-06-18 spec](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports),
still live in production everywhere — let a server "establish stateful
sessions." On `initialize`, the server *may* mint an `Mcp-Session-Id` header;
after that, the client "**MUST** include it on all subsequent HTTP requests,"
and a server that loses the session answers 404, at which point the client
"**MUST** start a new session." A session is server-side memory keyed by a
header. That memory is exactly what a load balancer cannot see.

You could already turn it off. The TypeScript SDK has long accepted
`sessionIdGenerator: undefined` to run stateless — and doing so
[dropped resumability](https://github.com/modelcontextprotocol/typescript-sdk/issues/340)
and half the server-to-client feature set with it. Stateless was the mode you
used when you gave up on features. The new spec inverts that: stateless is the
*default*, and the features were redesigned to survive it.

Three changes carry the weight.

**One: every request stands alone.** Per the announcement, each request now
"travels on its own, carrying its protocol version, client identity, and
client capabilities in `_meta`." There is no init handshake the server has to
remember. In the SDK, the server stops being a long-lived object bound to a
transport and becomes a handler that builds a fresh instance per request:

```typescript
// 2025: one server, one transport, session state in memory
new StreamableHTTPServerTransport({ sessionIdGenerator: undefined })

// 2026-07-28: a request handler, fresh instance each call
const handler = createMcpHandler(() => {
  const server = new McpServer({ name: 'my-server', version: '1.0.0' },
    { capabilities: { tools: {} } });
  // register tools/resources/prompts once
  return server;
});
```

That `handler` exports cleanly as a Vercel or Lambda function. No shared
process, nothing to keep warm, nothing to pin. Server identity itself moved
out of a stateful `serverInfo` handshake and into
`_meta['io.modelcontextprotocol/serverInfo']` on the response. Scale is now
someone else's problem — the platform's.

**Two: the server stops needing to talk first.** This is the part that made
sessions load-bearing in the old design. When a tool needed a value from the
user mid-call — elicitation — or wanted the client's model to sample a
completion, the server had to send a *server-to-client request*. Over HTTP
that meant holding open an SSE stream so the server had a channel to push
down. A held-open stream is a stateful connection by another name, and it dies
when the pod does.

The new spec replaces push with a return. The pattern is called Multi
Round-Trip Requests. A tool that needs input doesn't call out; it returns
`input_required` and waits to be called again with the answer:

```typescript
// 2025: server pushes a request down a live stream
const r = await ctx.mcpReq.elicitInput({ inputRequests: { count: schema } });
return handleResponse(r.count);

// 2026-07-28: server returns "I need input", client re-enters with it
const state = ctx.mcpReq.requestState<MyState>();
if (!state) {
  return inputRequired({
    inputRequests: { count: inputRequired.elicit({ /* schema */ }) },
    requestState: await codec.mint({ step: 'awaiting-count' }),
  });
}
const accepted = acceptedContent(ctx.mcpReq.inputResponses, 'count', SCHEMA);
return finalResult(accepted);
```

Sampling works the same way now — `inputRequired.sampling(...)` instead of a
pushed request. Roots and per-session logging as separate server-to-client
calls are simply
[gone](https://ts.sdk.modelcontextprotocol.io/v2/migration/support-2026-07-28);
log level rides along as a `_meta.logLevel` key on the request that needs it.
The through-line: nothing the server used to initiate needs a live back-channel
anymore. It answers, it forgets, it gets called again.

**Three: the state didn't vanish — it moved into the client's pocket.** A
multi-step tool call still has a middle. Where does the "we're on step two"
live if the server keeps nothing? In that `requestState` above. The server
mints it, the client carries it, the client hands it back on re-entry. It is
"**HMAC-signed, not encrypted**," minted with a secret key and a TTL. If that
smells like a signed cookie or a stateless JWT, it should — this is the oldest
trick in web scaling, and MCP just adopted it wholesale. Sign the state, give
it to the caller, verify it when it comes home, store nothing.

Two things follow from "signed, not encrypted," and you have to internalize
both. The flow state is *visible* to the client, so never put a secret in it.
And it is *tamper-evident with an expiry*, so a stale or forged `requestState`
must be rejected, not trusted. The SDK guide is blunt about the re-entry rule:
per-round `inputResponses` "replace on re-entry; never accumulate." You are
building a small state machine whose memory lives on the wire. That is more
discipline than a session hash-map, and it is the discipline that makes the
thing deploy.

## Who this is really for

Read the other changes and a pattern shows up. They are not for the person
writing a tool. They are for the box in front of it.

Header-based routing (SEP-2243) lets a tool mark an input as
`x-mcp-header: true`, and it arrives as an `Mcp-Param-*` HTTP header instead of
a field buried in the JSON-RPC body. The announcement is explicit about why:
"gateways can route and authorize on headers directly" without parsing the
body. Tool and resource lists gained `ttlMs` and `cacheScope` so a cache in the
middle knows how long to hold them. On the auth side, the
[2025 spec](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
already made the MCP server an OAuth 2.1 resource server that "**MUST** validate
that access tokens were issued specifically for them as the intended audience"
(RFC 8707) and forbade token passthrough — and it already required the
`Authorization` header "in every HTTP request from client to server, even if
they are part of the same logical session." The new spec finishes the job:
RFC 9207 issuer validation, and Dynamic Client Registration deprecated in favor
of Client ID Metadata Documents.

Per-request auth plus stateless transport plus header routing equals one thing:
an MCP server that is fungible behind a gateway. Any pod can serve any request;
the middlebox authorizes and routes without opening the envelope. That is the
protocol being shaped for infrastructure — the same layer where the platforms
sit, the one that ate LLMOps
([the wrapper is a feature, the gateway is a company](./2026-06-11-llmops-not-a-company.md)),
and the same tool list that already bloats your context window before you type
([the context tax](./2026-07-16-context-tax-before-your-prompt.md)). MCP hit
close to half a billion SDK downloads a month. At that size a protocol stops
being a spec and starts being a deployment target, and deployment targets get
optimized for the operators, not the authors.

## The honest cost

Stateless is not free, and anyone selling it as pure upgrade is skipping the
bill.

You gave up cheap resumability. A dropped connection mid-stream used to replay
from a `Last-Event-ID` cursor the server held; now a genuinely long-running job
belongs in the new `io.modelcontextprotocol/tasks` extension, which is
poll-based, or in your own store. The protocol stopped pretending the
connection was your database — which is correct, but it means *you* now own the
durability you were getting by accident. Every MRTR round also re-ships the
`_meta`, the capabilities, and the signed state, so a chatty multi-step tool
pays in round-trips and bytes what it used to pay in server RAM. For a plain
`tools/call` — already one POST, one response — you pay nothing new. For a
five-prompt wizard, you feel it.

The counter to the counter: stickiness was never free either. It was a hidden
coupling you stopped seeing because the SDK hid it, and it billed you the day a
pod recycled or the balancer rehashed. Moving the state onto a signed token
makes the cost legible and puts it where you can pay it deliberately. That is
the trade, stated plainly: you swap invisible server memory for visible,
signed, expiring wire state. It is the same trade the web made twenty years ago
when it moved from server sessions to signed cookies, and it scaled the web.

## Do, watch, ignore

**Do.** If you run a *remote* MCP server, move to the stateless handler now and
kill your session affinity — the `legacy: 'stateless'` default serves both 2025
and 2026 clients while you migrate, so this is not a flag day. Externalize any
per-call flow state into a sealed `requestState` (short TTL, no secrets in it),
or into a real store for anything long-running. And if you are not already
validating token audience with the RFC 8707 `resource` parameter, do that
regardless of this spec — it is the fix for the confused-deputy hole, not a
nice-to-have.

**Watch.** Header-based routing and CIMD-over-DCR are the tells for where this
goes: managed MCP gateways, per-tool authorization at the edge, caching
middleboxes. If your org runs more than a handful of internal servers, that
gateway is coming, and the servers that deploy behind it cleanly are the
stateless ones. Watch Tasks-extension adoption as the real home for
long-running agent work.

**Ignore.** The "stateless is slower" reflex — for ordinary tool calls it is a
wash, and the round-trip cost only bites deep wizards you can redesign. And
ignore this entirely if your servers are local `stdio` processes: that
transport is unchanged, no sessions, no gateways, nothing to migrate. This is a
remote-server story. If you ship one, the session era ended on July 28, and
your next deploy is lighter for it.
