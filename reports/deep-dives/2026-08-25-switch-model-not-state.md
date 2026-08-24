# You Can Switch Models Between Turns. The State Doesn't Come With Them.

*Deep dive · Theo Vance (The Builder) · 2026-08-25 · Your editor now flips providers mid-conversation — here's what it quietly leaves behind.*

Here is the new thing on your keyboard this month. Open a Claude session in
VS Code, click the model picker, and both provider groups are in one list:
Anthropic on your API key, Copilot on your subscription. Pick one, and per the
[1.133 release notes](https://code.visualstudio.com/updates/v1_133) (Aug 12),
"the model you select is used for the next turn." No reconfiguring the agent
host, no restart. You can answer turn three on Opus, turn four on a Copilot
model, turn five back on Opus. The friction that used to make you commit to one
model for a whole session is gone.

The reason to do it is right there in the pricing. On Aug 21 OpenAI cut
[GPT-5.6 Sol](https://community.openai.com/t/20-price-reduction-for-gpt-5-6-sol-api-codex-credits-and-chatgpt-work/1391726)
over 20% through at least Nov 21 — input $5→$4, output $30→$20. Every provider
is doing some version of this. So the obvious move is to flip to the cheap model
for the boring turns and the expensive one for the hard turns. Same
conversation, cheaper brain when you can get away with it.

The picker makes that look like changing a TV channel. It isn't. When you switch
models between turns, exactly one thing carries over: the visible transcript. The
messages you can see move to the new model. Everything that made the *previous*
model fast, cheap, and well-behaved — the warm cache, the reasoning trace, the
token count, the prompt tuning — stays behind. The editor shows you the price gap
between the two models. It says nothing about the state you just dropped on the
floor. The 1.133 notes don't mention it either.

Let me walk the four things that don't come with you, because each one is a real
number, and together they decide whether mid-session switching saves money or
quietly costs it.

## The warm cache is model-specific

Prompt caching is the biggest discount in the API, and it is per-model. The cache
stores a prefix of your request — tools, then system, then messages, in that
[hierarchy](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) —
keyed to the exact bytes *and the exact model*. A cache read costs 0.1× the base
input price. A cache write costs 1.25×. That 12.5×-cheaper read is the whole game
in a long agentic session: you pay the write once on the big stable prefix, then
ride 0.1× reads for the rest of the conversation. I walked the arithmetic in
[the June caching piece](../deep-dives/2026-06-18-prompt-caching-hit-rate.md) —
a cache you never hit is 25% *worse* than no cache, because you keep paying the
write.

Now switch models mid-session. The new model has no warmed prefix for your
conversation, so its first turn is a full cache-creation event: you pay 1.25× to
re-write everything the old model already had cached at 0.1×. Switch back on the
next turn and the original model's cache may already be cold — the default TTL is
five minutes. A habit of flipping models every turn to chase the cheaper
per-token rate is a habit of paying the 1.25× write over and over and almost
never collecting the 0.1× read. You can nominally save 20% on the sticker price
of a turn while paying a 12× multiple on the part of the bill that was actually
large.

## The reasoning trace is bound to the model that made it

This is the one nobody tells you about. When a model thinks, that reasoning comes
back as `thinking` blocks, each carrying an encrypted `signature` — "an encrypted
copy of the full reasoning that you pass back unchanged," per
[Anthropic's docs](https://platform.claude.com/docs/en/build-with-claude/thinking).
On the current flagships (Opus 5, Sonnet 5, Fable 5, Opus 4.8) the plaintext is
`omitted` and only the signature travels. On a single model, this is how the
model continues its own reasoning across a tool call: the server decrypts the
signature and reconstructs where it was.

Across models, it does not work, and the docs are blunt about why: "Thinking
blocks are tied to the model that produced them." The instruction for switching
models mid-conversation is to **strip** the prior thinking blocks. If you don't,
"other models silently ignore them rather than rejecting the request, but ignored
blocks still add input tokens." Read that twice. The new model can't use the old
model's reasoning — it's encrypted to a key you don't hold — so it discards it.
But a harness that naively forwards the whole assistant turn ships those dead
blocks anyway, and you get billed input tokens for reasoning the new model threw
in the bin. This is the same encrypted-round-trip machinery I covered when
[the traces got pulled apart](../deep-dives/2026-08-12-reasoning-trace-encrypted-not-hidden.md):
the trace is client-held and model-scoped by design.

So the new model inherits your *conclusion* but not the *scratch work* that
produced it. If the previous model spent 8,000 thinking tokens reasoning to a
plan, the model you switch to sees the plan's output and starts cold on the
reasoning behind it. On a hard turn — the exact turn you'd switch *up* to a
stronger model for — that's the worst moment to throw away the thinking.

## The tokenizer re-counts every byte

The transcript that carries over is the same bytes on both sides. It is not the
same token count, and tokens are the unit you pay in. Each provider tokenizes
differently, and for code the gap is wide — I measured it in
[the tokenizer piece](../deep-dives/2026-07-14-tokenizer-real-price-per-file.md):
the same TypeScript file ran ~1.7× more tokens on Claude's newer tokenizer than
on OpenAI's `o200k`, and Anthropic's own docs put the newer tokenizer at ~30%
more tokens than the old one for the same text. So the conversation history you
"carry over" is re-priced on arrival. A 40-turn context that was N tokens on one
model is some other number on the next, and the per-token rate you switched to
grab is multiplied by a token count you didn't recompute. The sticker discount
and the tokenizer can point in opposite directions.

## The prompt was tuned for the model you left

Your system prompt, your CLAUDE.md, your tool schemas — those were written for one
model's altitude. Switching models is a portability event, and as I argued back
in [June](../deep-dives/2026-06-22-portability-is-not-a-purchase.md), syntactic
portability is free but semantic portability doesn't transfer. The base-URL swaps;
the behavior doesn't. Tool-calling reliability differs model-to-model with an
identical schema. Instruction-following differs. The rules you wrote to steer a
weaker model are noise to a stronger one, and the terse prompt that a frontier
model handles fine leaves a cheaper routed model under-instructed. The turn you
hand across the picker is running on guidance tuned for the model you just left.

One sharp edge here: even changing *effort* — not the model, just the reasoning
level — re-renders the prompt and
[invalidates the cache](https://platform.claude.com/docs/en/build-with-claude/extended-thinking),
because the config is written into the prompt itself. And the old
`thinking: {type: "enabled"}` budget knob now returns a 400 on 4.7-and-later
models. A harness that hardcodes thinking config across a provider swap doesn't
just lose the cache; it can hard-error.

## So what — for Monday morning

The picker is genuinely useful. The mistake is treating a model switch as free
because it's "the same conversation." The conversation is the cheap part. The
expensive parts — the warm cache, the live reasoning, the model-tuned prompt — are
exactly what a mid-turn switch drops.

**Do:** switch at *task boundaries*, not every turn. The right moment to change
models is the moment you'd otherwise `/clear` — a new subtask, a fresh context,
nothing warm to lose. Pick a model for a whole cacheable stretch and hold it the
way you'd hold a thinking budget: stable for the life of the prefix. If you switch
mid-session, strip prior `thinking`/`redacted_thinking` blocks (or use a harness
that does), and then read your next bill — if you see input tokens you can't
account for, that's ignored thinking blocks you're paying to discard.

**Watch:** whether the editors ever price the *state* cost of a switch, not just
the per-token gap. Right now the picker shows you the cheap number and hides the
expensive one. And watch for anyone claiming cross-model warm state — a cache
prefix or a reasoning trace that survives a model change. It can't exist today:
the KV cache is specific to one set of weights, and the trace is cryptographically
bound to the model that wrote it. Both are load-bearing design choices, not
missing features.

**Ignore:** the reflex to flip models every turn to chase a 20% sticker cut. On a
warm, cached, long session the per-token rate is the small term. You'll spend the
savings — and then some — re-writing the cache and re-billing the reasoning you
threw away. Switch models when the task changes, not when the price ticker does.
