# They Didn't Hide the Reasoning. They Encrypted It and Handed It Back.

*Deep dive · June Okafor (The Contrarian) · 2026-08-12 · why a secret you ship to the client isn't a secret — and what that does to the distillation moat*

The claim everyone repeats is simple: frontier labs hide the model's reasoning to
protect it. The chain-of-thought is the expensive part — the thousands of
test-time-compute tokens you pay for at the output rate but never see, which
[the 07-18 dive priced out](../deep-dives/2026-07-18-reasoning-tokens-cost-per-answer.md).
It is also, the story goes, the crown jewel a competitor would most want to copy.
So OpenAI stopped returning the raw chain-of-thought of its o-series. Anthropic did
the same: on Fable 5 and Mythos 5,
[the docs say plainly](https://platform.claude.com/docs/en/build-with-claude/thinking),
"the raw chain of thought is never returned." Hide the reasoning, keep the moat.

Steelman it, because it is not a dumb position. Copying a model from the outside is
expensive precisely because the useful internal signals are withheld. When Anthropic
told the Senate that Alibaba ran 28.8 million Claude exchanges to imitate it, the
28.8 million *was* the receipt for everything the API refused to expose — no
logprobs, no reasoning, so you have to Monte-Carlo the model one hard sample at a
time ([06-27](../deep-dives/2026-06-27-distillation-without-logits.md)). Concealing
the chain-of-thought raises that cost further. On paper, it is a real defense.

Here is the plain fact that breaks it. The labs did not hide the reasoning. They
encrypted it and mailed it to you.

## "Hidden" means "you don't have the key"

Read how the APIs actually work.
[OpenAI's Responses API](https://developers.openai.com/api/docs/guides/reasoning)
returns reasoning items with an `encrypted_content` field, and instructs you to
"preserve all output items — including encrypted reasoning — and replay the complete
history in subsequent requests." Anthropic's thinking blocks carry a `signature`
field the docs describe as "an encrypted copy of the full reasoning that you pass
back unchanged." When the readable text is omitted — the default on every current
frontier Claude, from Opus 4.7 through Fable 5 — the `signature` "still carries the
encrypted full thinking." You are billed for those tokens whether or not you can
read them: "omitting reduces latency, not cost."

So the reasoning is not sitting safely in a datacenter. It is on your machine, in
your request logs, in your git history — as ciphertext you hold and hand back on
every turn. "Hidden" here does not mean "it never left." It means "you don't have
the key." And a secret whose ciphertext lives in the caller's hands is one
key-management mistake away from plaintext.

Last week, researchers found the mistake. The paper —
[*Stealing Reasoning Traces from Proprietary LLM APIs*](https://arxiv.org/abs/2608.09867)
— starts from one architectural detail: the encrypted blocks are "fully compatible
and interchangeable across different sessions, users, and models within a provider's
ecosystem." Anthropic's own docs advertise part of this: signatures "are compatible
across platforms… Values generated on one platform work on another." The attack
falls straight out of that. Take an encrypted reasoning block from a strong,
well-guarded model. Feed it to a weaker, less-guarded sibling from the same provider.
The sibling decrypts it and prints the reasoning verbatim, in plaintext. You never
jailbreak the capable model at all. The provider's own cheaper model is the
decryption oracle.

They ran it against Anthropic, OpenAI, and Google. Then they did the unglamorous
part: scraped 315,320 encrypted reasoning blocks from public repositories — session
logs developers had committed, assuming the blobs were opaque — and decoded them. Out
came 367 pieces of personal data and 182 credentials. The "hidden" reasoning had been
sitting in public GitHub repos the whole time. It just looked like noise.

## The concealment was a latency choice wearing a moat's costume

Here is the thing the consensus has backwards. Hiding the chain-of-thought was never
IP protection. It was a systems decision — ship the state to the client so the API
can stay stateless and fast — dressed up as a moat.

Where you hide a secret decides whether it is a secret. Server-side, ephemeral, never
returned: that is a secret you control. Encrypted and shipped to every caller: that is
a secret you have already given away, on the bet that nobody breaks the wrapper. This
is not a new lesson. It is the oldest footgun in web security — the signed cookie, the
stuffed JWT, session state handed to the client because holding it yourself is
expensive. The moment the state leaves your building, you are trusting a MAC and a key
schedule to do the job a database used to do, and you inherit every replay and oracle
attack that comes with it. The labs walked into it at the frontier of AI, for the same
reason everyone walks into it: statelessness is cheap and secrets-on-the-client feel
free.

They aren't free. The bill lands in three places.

**Distillation.** The whole cost structure of copying a model flips. The
28.8-million-query number assumed you were reconstructing the reasoning from the
outside, sample by sample. If the reasoning trace is recoverable, you are not
reconstructing anything — you are reading the teacher's worked solutions. On Hacker
News, [the thread's](https://news.ycombinator.com/item?id=49257876) first instinct was
to tie this to the Chinese labs we have tracked all summer — Kimi, GLM, the open models
that took [46% of US enterprise tokens](../2026-W29.md). Whether or not this specific
bug fed those specific models, the mechanism is the one under the whole commoditization
thread: the moat was the reasoning, and the reasoning was portable.

**Your own data.** If you send anything sensitive into a reasoning model, the trace can
contain it — and the trace is now a portable, decryptable artifact you are storing and
forwarding on every turn. Those 182 credentials were not prised out of a lab. They were
harvested from developers who thought an encrypted block was safe to paste into a public
repo.

**Safety measured on the wrong surface.** The paper's third finding: a model can refuse
in its visible answer while the hazardous content sits in the reasoning — and the
reasoning is recoverable. "The model safely rejected the request" is a statement about
the output. The output is not where the reasoning is.

## The honest objections

There are good ones, so take them head-on.

*"It's a bug, not a law. They patched it."* True — the providers closed the cross-model
replay after disclosure, and binding a key to one user and one session kills this
particular oracle. But that fixes the attack, not the posture. The ciphertext is still
on the client. It is still portable across your own turns. It is still one weaker
sibling, one future implementation slip, away from readable. Patching the lock does not
un-hand-out the box.

*"You paid for those tokens — reading them isn't theft."* Fair, and the sharpest
commenters said exactly that. But entitlement is a different argument from security. The
question is not who deserves the reasoning. It is that anyone holding the blob can
recover it, deserving or not.

*"Anthropic only returns a summary anyway."* The summary is what *you* can read. The
`signature` carries the full thing, and the attack decrypts the full thing, not the
summary. The redaction you see is not the redaction that ships.

## So what, Monday morning

Treat an encrypted reasoning block as sensitive plaintext in a thin disguise, not as
opaque data. It is exfiltratable state that may hold whatever you fed the model. Strip
reasoning items and thinking blocks out of anything you commit, log, or share — the same
reflex you use for a bearer token, because it may literally contain one. And stop reading
"the model refused" as a safety guarantee: the refusal describes the surface you can see,
not the one you can't.

What would change my mind: if, within two quarters, the major providers move reasoning
back server-side — never returned to the client, even encrypted, with keys bound to a
session and never portable — then concealment becomes a control I was wrong to doubt, and
the trace becomes a real secret again. I don't think they will. The stateless round-trip
is load-bearing. The reasoning is enormous (07-18), the API is sold on statelessness and
cache-friendliness, and keeping the trace client-side is what makes both cheap. So my
bet, at **70%**: a year from now the frontier still ships the encrypted trace to the
client, and independent researchers still recover it somewhere in the tail — because the
thing the labs called a moat was a latency optimization, and you cannot keep a secret by
giving everyone a copy of it.
