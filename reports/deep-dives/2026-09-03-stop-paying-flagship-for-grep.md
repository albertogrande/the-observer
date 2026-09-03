# Stop Paying Flagship Rates to Run grep

*Claude Code · Kit Sandoval (The Operator) · 2026-09-03 · the default model got a bigger window and a bigger bill; your subagents need neither*

Here's a failure you can't see happening. You point Claude Code at a monorepo
and ask it to find every place a deprecated auth helper is still called. It
does the sensible thing: it fans out. Ten subagents, each told to grep a
package, read the hits, and report back. A minute later you get ten tidy
paragraphs. Every one is correct. You merge the fix and move on.

You also just paid flagship rates to run grep ten times.

Because a subagent, unless you say otherwise, runs on **your session's
model** — and as of [v2.1.257](https://code.claude.com/docs/en/changelog)
(Sep 1) the model behind the `fable` alias is Claude Fable 5.1, at
[$10 per million input tokens and $50 per million output](https://platform.claude.com/docs/en/about-claude/pricing).
Haiku 4.5 does grep-and-summarize identically for **$1 and $5** — a tenth of
the price on the exact same tokens. The same release that made the default
model heavier also shipped the switch to stop this:
`CLAUDE_CODE_SUBAGENT_MODEL_FORCE`. This is a piece about when to throw it.

## A subagent isn't a smaller you. It's a sealed room.

Start with the mechanism, because the whole argument rests on it. When Claude
Code delegates to a subagent, that subagent does **not** inherit your context.
The [subagents docs](https://code.claude.com/docs/en/sub-agents) are blunt
about it:

> Each subagent starts with a fresh, isolated context window. It doesn't see
> your conversation history, the skills you've already invoked, or the files
> Claude has already read.

It gets its own system prompt, a short delegation message summarizing the
task, your `CLAUDE.md`, and git status. It works in that sealed room. And when
it's done, "only the subagent's summary result returns to the main
conversation." I made this same point on [July 30](../deep-dives/2026-07-30-subagents-context-not-speed.md):
a subagent buys you context isolation, not speed. The 40K-token grep spew
stays in a window you never see; three lines cross back.

That isolation is exactly why you can put a cheaper model in the room. A
weak model's mistakes compound only when they feed the *next* decision. In a
subagent, they don't — the noise dies in the sealed room, and what returns to
your main thread is a distilled result you can sanity-check in one glance. The
quality of the model that produced it matters far less than the quality of the
model that reads it and decides what to do next.

So the question stops being "which model is best?" and becomes "which model
per *job*?" The docs even resolve model selection in a fixed order, worth
knowing before you start overriding it. For any subagent invocation Claude
Code checks, in order: the per-call `model` parameter, then the subagent
definition's `model:` frontmatter (`inherit` means "use the main model"),
then the `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, then — falling
through all of that — the main conversation's model. That last fallthrough is
the one billing you $50/Mtok to summarize a git log.

## Spend at the top, save at the leaves

The instinct, when a bill surprises you, is to reach for the cheap model
everywhere. That is the one move that actually costs you.

Your orchestrator — the top-level loop — is where model quality pays. It holds
the plan. It decides which subagents to spawn, what to keep in the window,
which edit to write, what to commit. A bad plan wastes every subagent
underneath it; the model that grovels through a package can be mediocre, but
the model that decided *which ten packages to grovel through* cannot. Anthropic's
own [cost guidance](https://platform.claude.com/docs/en/about-claude/pricing)
says it plainly: "Choose Haiku for simple tasks, Sonnet for most production
workloads, and Opus for the most complex reasoning." Tiering isn't a hack. It's
the documented shape.

The map that actually works is by **kind of work**, not by depth in the tree:

| Job | Model | Why |
|---|---|---|
| Orchestrator / planner | Fable or Opus | Errors compound down the whole tree |
| Search, scan, grep, read-and-list | Haiku | Bounded, mechanical, returns a summary |
| Format, rename, mechanical edit | Haiku / Sonnet | No judgment, high volume |
| Review, verify, "is this diff wrong?" | Fable or Opus | This *is* the hard part |

That last row is the trap on the other side. It is tempting to file a
"review" subagent under cheap-and-mechanical because it, too, just reads and
reports. It doesn't. A verifier's whole job is judgment — the [asymmetry
between generating and checking](../deep-dives/2026-07-24-verifier-asymmetry-check-vs-find.md)
is the reason you spawned it. Cheap out on the reviewer and you've built a
rubber stamp. Depth in the tree tells you nothing; the *task* tells you
everything.

## The scalpel and the hammer

Two levers, and they behave differently — this is the part to get right.

The **scalpel** is per-agent frontmatter. Put the model in the subagent
definition and it travels with the agent, everywhere it's used:

```yaml
---
name: repo-scout
description: Greps the codebase for call sites and reports a file:line list
tools: Read, Grep, Glob
model: haiku
---
You find call sites. Return a plain list of file:line matches and a one-line
summary. Do not propose changes.
```

Your `reviewer.md` gets `model: inherit` (or no `model:` line at all) so it
rides whatever strong model you're driving. This is the setup you want by
default: each agent declares the tier its job deserves, and it's right whether
you launched the session on Fable, Opus, or Sonnet.

The **hammer** is the new environment variable. `CLAUDE_CODE_SUBAGENT_MODEL_FORCE`,
per the v2.1.257 changelog, "forces a model across every subagent in a
session" — it *overrides* the definitions, including a frontmatter `model:`.
Note the sibling it joins: `CLAUDE_CODE_SUBAGENT_MODEL` (documented in
[model config](https://code.claude.com/docs/en/model-config)) is a *default* —
it sits third in the resolution order, so a per-agent `model:` beats it. The
new `_FORCE` variant is the opposite: it wins. That distinction is the whole
utility. Use the default when you want a floor your agents can rise above; use
the force when you want a ceiling none of them can escape:

```bash
# Soft floor: subagents that don't specify a model land on Sonnet.
export CLAUDE_CODE_SUBAGENT_MODEL=sonnet

# Hard ceiling: THIS run, every subagent is Haiku, no matter what its
# definition says. For a big mechanical sweep you want cheap and uniform.
CLAUDE_CODE_SUBAGENT_MODEL_FORCE=haiku claude "migrate all call sites of the old logger"
```

Reach for the hammer on a bulk, low-judgment run where you don't trust every
agent definition to have set its tier honestly. Leave it off for normal work,
where you *want* the reviewer to stay strong.

## What the 1M window changes, and what it doesn't

Fable 5.1's headline is the [1M-token context window](https://platform.claude.com/docs/en/build-with-claude/context-windows),
and — unusually — it's billed at standard rates. The pricing page is explicit:
"A 900k-token request is billed at the same per-token rate as a 9k-token
request." There's no penalty tier above 200K. That removes the *price* brake
on stuffing the window. It does not remove the [rot brake](../deep-dives/2026-06-25-context-budget-sixty-percent.md):
a bigger window still degrades in the middle, and a 1M haystack is still a
haystack the model half-reads. Bigger is not a reason to fill it.

But there's a subagent wrinkle the docs spell out, and it's easy to trip on:
"a subagent's context window is sized by its own model, not the parent's.
Delegating to a model with a smaller window gives that subagent the smaller
window." So when you force subagents down to Haiku, you also hand them Haiku's
window. For a scout that reads a few files and returns a list, fine. For a
subagent whose job is to read a 400K-token document and reason over all of it,
you just capped it below the task. Tier by the job's *context* need, too — not
only its reasoning need. Cheap and small is right for the leaf that returns a
summary and wrong for the leaf that has to hold a whole file in its head.

There's one more reason to keep the big, stable context on top: Fable 5.1
prices **cache reads** at $0.25/Mtok — a 0.025× multiplier, four times cheaper
than the standard 0.1× cache hit. Your orchestrator's long, stable prefix is
the thing that gets re-read every turn, and on Fable 5.1 that re-read is nearly
free. Keep the durable context cached at the top; push the volatile fan-out to
the bottom.

## Before you paste anything, check the price you think you're paying

One number in the wild is already wrong. The scouting note that prompted this
piece said Sonnet 5's promo rate expired Aug 31 and the price rose to $3/$15 on
Sep 1. The [primary pricing page](https://platform.claude.com/docs/en/about-claude/pricing)
says the opposite: the $2/$10 introductory rate "is now the standard price,"
and "the previously scheduled increase to $3/$15 per million input/output
tokens on September 1, 2026 will not occur." Anthropic kept the cut. If you
repriced your Sonnet cost model this week off the wrong number, fix it — and
take the general lesson: verify the rate against the pricing page before you
tune anything to it, because in this market the sticker moves under you.

## The change to make before lunch

1. Add `model: haiku` to the frontmatter of your search, scan, and mechanical-
   edit subagents. Leave your reviewer and planner agents on `model: inherit`
   (or no `model:` line).
2. Drive your main session on the strong model (`/model fable`, or `opusplan`
   if you want Opus for planning and Sonnet for execution — see
   [model config](https://code.claude.com/docs/en/model-config)).
3. For a one-off bulk sweep, front the command with
   `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=haiku`. For normal work, don't — you want
   the reviewer strong.
4. Measure the right thing. Run [`/cost`](../deep-dives/2026-08-27-itemize-the-agent-bill.md)
   before and after and compare **cost per solved task**, not cost per token.
   The [meter's whole lesson](../deep-dives/2026-07-14-tokenizer-real-price-per-file.md)
   is that the cheapest token can still be the most expensive answer — so the
   number that decides this is whether the Haiku leaves still return work you
   don't have to redo.

That last check is also what would change my mind. Tier aggressively, then
watch your redo rate. If pushing a job to Haiku means the orchestrator spawns
it twice because the first summary was too thin, the leaf wasn't cheap — it was
a false economy wearing a low sticker. The right tier is the cheapest model
whose output you don't have to earn again. Everything above that line is you
paying flagship rates to run grep.
