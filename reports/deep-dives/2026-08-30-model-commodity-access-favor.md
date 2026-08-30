# The Model Is a Commodity. The Right to Call It Isn't.

*Deep dive · June Okafor (The Contrarian) · 2026-08-30 · OpenAI is cutting Cursor off by November 12 — and the lesson isn't about Musk.*

On Friday, OpenAI [told SpaceX it will wind down Cursor's model access by
November 12](https://openai.com/index/our-decision-on-cursor-following-its-acquisition-by-spacex/),
citing terms-of-service violations by Musk's *other* companies — xAI, X, the
lot. SpaceX only [closed its ~$60B purchase of Cursor's parent, Anysphere, on
August 14](../2026-W34.md). So the sequence is clean and a little absurd: a
coding tool with more than a million daily developers gets ten weeks to
replace one of its model suppliers, and the reason has nothing to do with the
tool, its usage, its bill, or its users. It has to do with who signed the
acquisition papers.

There are two easy reads, and most people are picking one of them. The first:
this is a billionaire feud, Altman versus Musk, round nine — theater, not
signal, safe to ignore. The second is quieter and more dangerous, because
it's the one thoughtful engineers actually believe right now: *models are
commoditized now, so this doesn't matter.* Stripe [bought OpenRouter for more
than $7B](../2026-W33.md) three weeks ago; there are 400-plus models behind
one OpenAI-compatible endpoint; if a provider cuts you off, you change a
base URL and move on. Cursor is model-agnostic by design. Losing one supplier
is a config change.

I want to steelman that second read all the way, because it's mostly true —
and then show you the exact place it breaks, because the break is the whole
story.

## The part everyone gets right

The token really is fungible. A year of price wars and open-weight floods —
GLM, Qwen, DeepSeek, Kimi — drove frontier-grade coding capability to a
commodity. The benchmarks that used to separate models now [sit inside each
other's error bars](../2026-W34.md). The plumbing standardized on OpenAI's API
shape, so swapping providers is, at the wire level, a URL and a key. Cursor
itself routes across Anthropic, OpenAI, and its own house models. On paper,
this is exactly the architecture the portability gospel told you to build. If
capability is a commodity and the interface is standard, no single supplier
should be able to hurt you.

So here is the uncomfortable question. Cursor did everything right. It is
model-agnostic. It has alternatives. And it still got a hard November 12
cutoff it cannot appeal, for a reason it cannot fix, because a reason it
cannot fix is the *point* — the trigger was an acquisition, not a behavior.
If commoditization made supplier risk go away, this cutoff could not exist.
It exists. So commoditization did not make supplier risk go away. It moved it.

## What it moved to

Here's the inversion. Commoditization made the *token* fungible and, in the
same motion, concentrated the *serving* of that token into a handful of
suppliers — each of whom can revoke your access for reasons that have nothing
to do with price, quality, or how you use the API. The scarce, non-portable
asset was never the weights. Anyone can rent equivalent weights this
afternoon. The scarce asset is a specific counterparty's ongoing willingness
to serve *you* — and that willingness is a business and political variable,
not a technical one. You do not own your model access. You rent it, from
someone who is allowed to leave.

We have watched this movie before, at least twice, with receipts.

In January 2023, third-party Twitter clients [stopped working within a single
day](https://techcrunch.com/2023/01/16/twitters-third-party-client-issue-is-seemingly-a-deliberate-suspension/).
No warning. A week later the developer agreement was quietly rewritten to
prohibit building "a substitute or similar service." Tweetbot — twelve years
old — [shut down](https://tapbots.com/tweetbot/), along with Twitterrific and
two dozen other clients. These were not misbehaving apps. They were paying,
long-standing developers who had built on a platform that was "open" right up
until the platform's owner decided it wasn't. The notice period was measured
in hours.

Five months later, Reddit repriced its API to [about $0.24 per thousand
calls](https://techcrunch.com/2023/05/31/popular-reddit-app-apollo-may-go-out-of-business-over-reddits-new-unaffordable-api-pricing/)
— $12,000 per fifty million requests. Apollo, the best third-party Reddit
client, was making seven billion requests a month. That worked out to roughly
$20 million a year, against about $500,000 in revenue. Christian Selig got
around thirty days and shut Apollo down on June 30. Again: the developer's
behavior didn't change. The platform's math did. The API was a favor, and the
favor was withdrawn.

The pattern is the same each time, and it is not about villains. A platform
opens an interface to grow. Builders build on it. Then the platform's
incentives shift — a new owner, a competitive conflict, a margin problem, a
policy — and the interface that felt like infrastructure turns out to have
been a terminable contract all along. What kills you is never the thing you
were watching. It is the counterparty you didn't think of as a counterparty.

## Why commoditization makes this worse, not better

You'd expect a world of interchangeable models to be a *safer* world to build
in. It isn't, and the reason is precise.

When capability was scarce, you were locked to the best model and you *knew*
you were locked. The dependency was visible; you priced it. Now that a dozen
models are good enough, the lock-in feeling evaporates — and that feeling is
the trap, because while capability spread out, serving concentrated. The
thing you genuinely cannot get somewhere else is not "a good model." It is
*this supplier's contract*, and that is exactly the thing an acquisition, a
feud, or a ToS clause can void.

And the "just swap the base URL" reflex is doing a lot of load-bearing work it
can't hold. Syntactic portability is free; [semantic portability is
not](../2026-06-22-portability-is-not-a-purchase.md). The same prompt does not
behave the same across two models — instruction-following differs,
[tool-calling reliability differs at an identical schema](../2026-08-25-switch-model-not-state.md),
the warmed cache is gone, the tokenizer re-counts the bill. Cursor doesn't
have to change a URL. It has to re-tune and re-eval the prompt and
tool-calling surface of an entire product on a different provider, and prove
it didn't regress for a million daily users, inside ten weeks. "A config
change" is what this looks like from the outside. From the inside it is a
migration with a deadline someone else set.

## The counter-thesis

So I'll commit to the claim the two easy reads both miss: **model
commoditization does not remove supplier risk — it relocates it, from a
capability question you have already solved to a governance question you
haven't.** The old question was *is the model good enough?* That's answered;
they mostly are. The new question is *will this counterparty keep serving me,
and can I actually cut over inside whatever notice window they choose to
give?* That question has no technical answer, and almost nobody is provisioning
for it.

The only hedge that pays is the one this publication has argued since June:
[portability is an eval discipline, not a purchase](../2026-06-22-portability-is-not-a-purchase.md).
A second provider that is *actually wired and continuously eval'd* on your real
workload — one you could switch to this afternoon and know your quality held —
is worth something. A second provider that exists as a line item in a "we
support multiple models" slide is worth nothing on the day you need it,
because that's the day you discover the re-tune you never did. Cursor is the
proof that finally moved this from a hypothetical to a bill: the revocation
happened, at the application layer, to a well-run company that had options.

Let me take the strongest objections head-on.

*"This really is just Musk. A normal customer never gets cut."* Maybe you
never get cut over a billionaire feud — but the feud is the trigger, not the
mechanism, and the mechanism generalizes. The Apollo and Tweetbot developers
weren't feuding with anyone. They were cut when the platform's math changed.
Acquisition is one trigger; a price change, a competitive-conflict clause, a
policy shift, a capacity crunch are others, and every major model provider's
terms of service already reserve broad discretion to suspend access. The
question isn't whether *your* reason will be as colorful as Musk's. It's
whether you've assumed a supplier can't leave.

*"Cursor will just move to Anthropic and its own models. No big deal."* Partly
true — and it proves the point rather than defusing it. Cursor *can* absorb
this because it is large and had alternatives already running. A smaller shop
built on one provider's assistant API has no such runway; its ten weeks are
spent discovering that its prompts don't port. The cost was never the base
URL. It was the eval you didn't run.

*"Multi-provider is a lowest-common-denominator tax you'll probably never
need."* This is the real tension, and I won't wave it away — carrying a warm
second provider costs engineering you might spend zero times. But the tax is
small and now the tail is *demonstrated*, not imagined. You only have to
decide which slice of your product genuinely can't go dark for ten weeks, and
wire a fallback for that slice. The rest can stay single-sourced. Tiered, not
total.

## So what, on Monday

If you build on a model API, do four concrete things this week. Map the
single-supplier chokepoints in your agent stack that you do not control — not
"which models could we use," but "which one, today, would take us down if it
vanished." For the slice that can't go dark, keep one alternate provider
*wired and eval'd* on your actual workload, and run a real cutover drill, not
a checkbox test — you're measuring whether your prompts and tool calls survive
the move, because that's what fails. Read "multi-model support" in any vendor
pitch, including your own roadmap, as marketing until the drill passes. And go
read your provider's ToS suspension clause and notice terms, so you know
whether you'd get Cursor's ten weeks or Tweetbot's ten hours.

What would prove me wrong: if, by the end of 2027, no other comparably sized
AI developer tool loses model access for counterparty or business reasons —
not usage, not payment — then Cursor was a genuine one-off, a billionaire
spat with no structural tail, and provisioning for supplier revocation was
wasted motion. I don't think that's where this goes. When capability
commoditizes, the margin the survivors fight over moves to who controls the
serving — and control means the right to say no. I'd put it at **70%** that we
see at least one more such cutoff before 2027 is out. The model is a
commodity. The permission to call it is the product now, and it was never
yours to keep.
