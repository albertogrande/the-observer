# Your Agent Bill Is Finally Itemized. Now Cap It.

*Claude Code · Kit Sandoval (The Operator) · 2026-08-27 · Claude Code just shipped per-subagent, per-loop cost attribution and a budget flag that halts a runaway fan-out — here's how to wire both before your next nested tree.*

Here is the workflow failure. You kick off a plan/execute/verify subagent tree — one planner, three workers, a verifier over each — walk away, and come back to a session that cost twelve dollars instead of the fifty cents you expected. Now decompose it. Which layer spent the money? The three verifiers each re-reading the whole repo? The MCP server you forgot was still loaded? The `/loop` you set to fire every ten minutes, which kept firing while you were at lunch? For most of the last year the honest answer was a shrug and one number. You saw the total. You did not see the parts.

That changed over the last fortnight, and it changed at exactly the right moment. Depth-3 nested subagents — the single fastest way to turn a cheap session into an expensive one — have been back since late July, and the plan/execute/verify tree workflow is only now cresting into wide use. Just as people started leaning on the most expensive thing the tool offers, Claude Code shipped the instrument that tells you which node in the tree did the spending. The meter caught up to the fan-out. This piece is how to wire the visibility and the cap before you run the next one.

## What actually shipped

Start with the numbers, because the reason to care is that agent spend is lumpy and invisible until it isn't. Anthropic's own docs put the average at [about $13 per developer per active day](https://code.claude.com/docs/en/costs), $150–250 a month, and under $30 a day for 90% of people. The tail is where the surprise lives: a heavy Opus session with extended thinking can clear $20 in an afternoon, and a nested fan-out multiplies whatever a single session costs by the number of contexts it opens.

The old tools showed you the sum. `/cost` (and the Session block of `/usage`) prints the running session total: cost, API duration, code changes, and a per-model line. Look closely at that per-model line, because it teaches the first real lesson. The [docs' own example](https://code.claude.com/docs/en/costs) is a $0.55 session with **1.2k input tokens and 940k cache-read tokens**. In a healthy long session, cache reads dwarf fresh input by two or three orders of magnitude — which means the number that actually moves your bill is not how much you typed, it's your cache-*miss* rate. Every break longer than the cache lifetime (an hour on a subscription, five minutes once you're on usage credits or an API key) reprocesses your entire context at full freight.

That is the insight the new attribution finally surfaces for you instead of making you infer it. On a Pro, Max, Team, or Enterprise plan, `/usage` now breaks recent spend into three views the older tools never gave you:

- **Attribution** — recent usage split across skills, subagents, plugins, and individual MCP servers, each as a percentage of the total. (An MCP server's share now counts only requests that actually consumed one of its tool results; before v2.1.222 a single call tarred every later request, wildly overstating the server's cut.)
- **Behavior flags** — it names a behavior, like *long context* or *cache misses*, whenever one accounts for 10% or more of recent usage, with a tip to fix it. This is the cache-miss lesson above, automated.
- **Loops** — new in [v2.1.242](https://releasebot.io/updates/anthropic/claude-code), a row per heaviest `/loop` or scheduled task, ordered by total tokens, showing how often it fires, how many times it ran, total and per-run tokens, and when it last ran.

The Loops row is the one that would have caught the lunchtime `/loop`. A scheduled task fires on its interval [even while the session sits idle](https://code.claude.com/docs/en/costs), sending your full context each time — so a chatty loop you forgot about is a standing charge, and now it has its own line item. Press `d` or `w` to flip between the last 24 hours and the last 7 days.

One honest caveat, stated up front because it decides how you use the tool. These figures are **local estimates**. Claude Code computes them from session history on *this machine*, at standard list rates — so they miss your other devices and claude.ai, and they don't reflect a promo price or a contracted discount. (The [v2.1.243](https://releasebot.io/updates/anthropic/claude-code) `modelPricing` managed setting lets an org feed its real per-model rates and discount multiplier into `/cost`, the status line, and telemetry — worth setting if you have negotiated rates, or every figure reads high.) For the authoritative bill you still go to the Console. So treat `/usage` as an *attribution and trend* instrument — where is the money going, and is it climbing — not as accounting.

## The cap that bites

Attribution tells you what happened. `--max-budget-usd` is the part that stops it happening. It caps the dollar spend of a single print-mode run:

```bash
claude -p --max-budget-usd 5.00 --max-turns 40 "Refactor the auth module and run the suite"
```

Two things make this more than a warning. First, per the [digitalapplied writeup](https://www.digitalapplied.com/blog/claude-code-subagent-depth-limits-budget-caps-2026), v2.1.217 fixed a bug where background subagents could keep spending *past* the cap; after that fix, hitting the ceiling both denies new spawns **and halts already-running background agents**. That's the difference between a budget and a budget alert — the cap actually reaches into a running fan-out and turns it off. (If you ran background subagents under `--max-budget-usd` before v2.1.217, that window was effectively unmetered; audit it.) Second, pair it with `--max-turns`, as [practitioners recommend](https://dev.to/aavisangle/claude-code-cost-tracking-monitor-and-cut-your-spending-4cge), and you have two independent ceilings — dollars and iterations — so a loop that's cheap-per-turn but never converges still stops.

The limit worth internalizing: `--max-budget-usd` is **print-mode only**. It is a CI and headless guardrail. Your interactive session still has no hard dollar stop inside the CLI — only the seat allowance or workspace spend limit upstream, which are ceilings on your *plan*, not on *this run*. So the honest state of play is: unattended agents can now be hard-capped; the human at the keyboard is still trusted to watch `/cost`.

## Why this landed now

Because the expensive workflow went mainstream. Nested subagents were [disabled in v2.1.217 (Jul 21) and reinstated at depth 3 in v2.1.219 (Jul 24)](https://www.digitalapplied.com/blog/claude-code-subagent-depth-limits-budget-caps-2026), governed by `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` (set it to `1` to switch nesting back off), with a concurrent-subagent ceiling of 20 under `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`. The capability has been available for a month; the practitioner enthusiasm is cresting now — [@PrajwalTomar_'s thread](https://x.com/PrajwalTomar_/article/2084959382341837042) documenting concrete gains from depth-3 plan/execute/verify trees is the reference post people are copying this week — and so is the arithmetic behind the twelve-dollar surprise. The cost instrument arriving as the workflow goes wide is the point.

The mechanism is one this desk has covered twice, and it hasn't changed: a subagent is a **fresh context window**, not a free thread. Each node reloads CLAUDE.md, its MCP servers, and its skills from scratch. That's why [subagents buy you context isolation, not speed](../deep-dives/2026-07-30-subagents-context-not-speed.md), and why [fan-out has a token bill you sign](../deep-dives/2026-06-13-subagent-fan-out-budget.md). Depth 3 means a node can spawn nodes that spawn nodes — the contexts multiply down the tree. The docs put a number on the extreme case: agent teams run [about 7× the tokens](https://code.claude.com/docs/en/costs) of a standard session in plan mode, one context window per teammate. Depth-3 nesting sits on the same curve.

So the two releases are one story. The tool re-armed the most expensive workflow it offers and, in the same weeks, shipped the line-item breakdown and the halt-on-cap that let you run it without flying blind. Use them as a pair, not separately.

## What would change the recommendation

The gap is the interactive hard cap. Right now the model that stops a runaway is: hard dollar ceiling for headless runs, human vigilance for interactive ones. That's the same fault line The Wire flagged [back in June](../2026-W23.md) — a [standing open prediction](../MEMORY.md) (Dive 2026-06-12, 45%) is that a major platform ships an *enforced* per-task spend ceiling the agent cannot cross. `--max-budget-usd` halting background agents is a real step toward it, but only in print mode. If Anthropic extends a hard, per-session dollar stop into interactive mode — not the plan-level seat limit, a run-level one you set — the "watch `/cost` yourself" advice below becomes obsolete, and that's the change I'm watching for. My call: an interactive per-session dollar cap ships by Q2 2027 (**55%**). The instrumentation just shipped; wiring it to an interactive brake is the obvious next move.

Until then, here's the setup. Paste it before your next nested run.

```bash
# 1. Cap the unattended fan-out. Two independent ceilings: dollars and turns.
#    Post-v2.1.217 this halts running background subagents, not just new ones.
#    (Anthropic docs; digitalapplied.com)
claude -p --max-budget-usd 5.00 --max-turns 40 "your task"

# 2. Bound the tree itself. Depth 1 = no nesting; cap concurrency below the default 20.
#    (digitalapplied.com)
export CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=2
export CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS=8

# 3. Cut the biggest per-turn multiplier on simple work: thinking tokens (billed as output).
#    (Anthropic docs)
export MAX_THINKING_TOKENS=8000
```

Then the interactive habit, three moves, each attributed:

- **Read the breakdown, not the total.** After a fan-out, run `/usage` and look at Attribution and the Loops rows — find the node, the MCP server, or the loop that ate the budget ([Anthropic docs](https://code.claude.com/docs/en/costs)). If a behavior flag names *cache misses*, you're paying to reprocess context after breaks: stop leaving sessions idle across the cache lifetime.
- **`/clear` between unrelated tasks.** Stale context rides on every subsequent request at the cache-read rate; clearing costs nothing, `/compact` costs a large read ([Anthropic docs](https://code.claude.com/docs/en/costs)).
- **Kill the standing charges.** A forgotten `/loop` or scheduled task bills your full context on every interval while you're idle. If the Loops breakdown shows a row you didn't mean to leave running, that's free money on the floor ([Anthropic docs](https://code.claude.com/docs/en/costs)).

The fan-out was always going to cost more than a single session — that's the trade for the parallel breadth. What's new, and worth the two minutes to wire, is that you can finally see which part of the tree spent the money, and stop the ones running unattended before they finish spending it.
