# Delete the Rules You Wrote for a Dumber Model

*Deep dive · Theo Vance (The Builder) · 2026-07-26 · Anthropic cut 80% of Claude Code's system prompt and lost nothing — so what should you cut from yours?*

You have a `CLAUDE.md`. You've been growing it since spring. Every time the agent
did something dumb, you added a line to stop it: *no comments unless asked. Prefer
async. Never edit the lockfile by hand. Always run the tests before you say you're
done.* It's a couple hundred lines now. It works. You'd defend every rule in it.

This weekend a more capable model shipped, and that pile of rules is quietly
working against you.

The peg is two Anthropic releases in two days. On Friday, [Claude Opus 5](https://www.anthropic.com/news/claude-opus-5)
landed on every platform — same $5/$25 per-Mtok price as Opus 4.8, but hitting
within 0.5% of Fable 5 on CursorBench at, by Anthropic's own framing, half the
cost per task, and "much stronger at verifying its work and iterating carefully."
On Saturday, Anthropic published [the new rules of context engineering](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models)
for the Claude 5 generation. The load-bearing sentence: *"We removed over 80% of
Claude Code's system prompt for models like Claude Opus 5 and Claude Fable 5 with
no measurable loss on our coding evaluations."*

Read that again as a builder. The team that ships the agent you use deleted most
of the instructions they'd written for it, ran their evals, and nothing got worse.

## A rule is compensation for a capability gap

Here's the exact before/after they published. The old system prompt said:

> Default to writing no comments. Never write multi-paragraph docstrings or
> multi-line comment blocks — one short line max.

The new one says:

> Write code that reads like the surrounding code: match its comment density,
> naming, and idiom.

The old line is a blunt proxy. When the model couldn't reliably judge how much to
comment, you handed it a rule that always errs one way: none. That was the right
trade for a model that would otherwise dump a paragraph over every function. But a
proxy overrides judgment in *both* directions. On a model that can read the room,
"never write a multi-line comment" now stops it from writing the docstring that
was actually the right call — because the file it's editing is full of them.

That's the general shape. **A rule you write is human knowledge you bolt on to
compensate for something the model can't do yet.** It helps a weak model and taxes
a strong one. The tax is invisible until the model is good enough that its own
judgment would have beaten your rule.

If that sounds familiar, it's [the bitter lesson](http://www.incompleteideas.net/IncIdeas/BitterLesson.html)
arriving in your prompt file. Rich Sutton's 2019 essay is about the same pattern
one level down: general methods that scale with computation keep beating
hand-engineered human knowledge, and researchers keep learning it the hard way
because the hand-engineering feels like progress right up until it caps you.
Context engineering is relearning it now. Every rule in your `CLAUDE.md` is
hand-engineered knowledge. The better the model, the more of it flips from help to
handcuff.

## What actually changed, and by how much

Be precise about the 80%. Anthropic doesn't publish what the base was, and "system
prompt" is a slippery unit. Community trackers like [Piebald's](https://github.com/Piebald-AI/claude-code-system-prompts)
split Claude Code's preamble into the *instruction* prompt (the rules — low
thousands of tokens) and the *tool schemas* (much larger). As [the 07-16 context-tax
dive](./2026-07-16-context-tax-before-your-prompt.md) measured, the ~33k tokens
Claude Code sends before your first keystroke are mostly tool definitions, not
prose rules. So the 80% is 80% of the hand-written *instructions* — the rules —
not 80% of the whole preamble. It's a checkable claim (the diffs are public,
version by version) on their own coding eval. Worth verifying before you copy the
move; not worth dismissing as marketing.

And it isn't a launch-day pivot. Anthropic's [September 2025 guidance](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
already told you to find "the smallest possible set of high-signal tokens" and
warned about the "Goldilocks zone" between two failure modes — brittle hardcoded
logic on one side, vague guidance that "falsely assumes shared context" on the
other. The "right altitude" was already the target a year ago. What Opus 5 does is
*move the altitude up*: the more the model can infer, the more of your specific
rules you can pull out from under it. This is a trend line, not a reversal.

Four more shifts from the post, translated into things you change:

- **Examples → expressive interfaces.** Anthropic found that "giving examples
  actually constrains them to a certain exploration space." Instead of few-shotting
  a tool call with three sample inputs, design the parameter so the shape teaches
  itself — a `status` enum of `in_progress | blocked | done` says more, and boxes
  the model in less, than a worked example. This is the same lesson as [the 07-05
  tool-schema dive](./2026-07-05-tool-schema-off-distribution.md) from the other
  direction: the schema, not the prose, is where reliable tool use lives.
- **Progressive disclosure → skills.** Move situational guidance out of the
  always-on system prompt into [skills](./2026-07-09-skill-costs-one-sentence.md)
  the model loads only when the task calls for them. The rule that only matters
  when you touch the migration folder shouldn't be billed on every turn.
- **Kill repetition.** Older setups stated the same instruction in the system
  prompt *and* the tool description. The new models need it once — in the tool
  description.
- **Auto-memory over manual saves.** Claude "now automatically saves memories that
  are relevant to the work and to you," instead of waiting for you to hand-edit
  `CLAUDE.md`.

## The line you must not cross

Here's where a Builder has to stop nodding along. "Fewer rules, trust the model"
is exactly the advice a vendor wants you to take, and taken literally it's
dangerous. Less prescription means less determinism. In a regulated pipeline, a
security boundary, or anything touching prod, you *want* hard constraints, and no
amount of model capability changes that.

The resolution is that you've been keeping two very different things in the same
file, and only one of them is scaffolding.

- **Instructions that compensate for capability** — "no comments," "prefer async,"
  "always run the tests." These are tutoring. Delete them on a capable model; its
  judgment is now better than your proxy.
- **Constraints that bound blast radius** — deny `curl`, run in the sandbox, never
  push to `main`, don't read `~/.aws`. These are brakes. A stronger engine needs
  the *same* brakes, not fewer. The [PreToolUse hook](./2026-07-02-hooks-are-the-real-guardrail.md)
  and the [OS sandbox](./2026-07-23-sandbox-is-the-real-brake.md) aren't
  instructions the model can outgrow; they're limits on what it's *allowed* to do,
  and capability is orthogonal to permission.

The bitter lesson deletes the tutor. It does not delete the seatbelt. Don't
confuse "the model can judge this" with "the model is allowed to do this."

And keep your gotchas. The post is explicit: "Keep your `CLAUDE.md` lightweight
and briefly describe what your repo is for, but spend most of the tokens on
gotchas inside of the codebase." A gotcha is a non-derivable fact — this service's
weird retry semantics, that one flag that means the opposite of what it says. The
model can't infer it by reading the repo, so it isn't scaffolding, it's
information. Cutting rules is not cutting facts.

## Do / watch / ignore

**Do, Monday morning.** Open your `CLAUDE.md` and your agent's system prompt. Go
line by line and ask one question: *would a competent engineer who just read this
repo already do this?* If yes, it's scaffolding — cut it. Keep only two kinds of
line: non-derivable gotchas, and hard safety constraints. Then do the part
Anthropic did and most people skip: run an eval fixture before and after. You
can't refactor the thing that produces your output without [a check on the
output](./2026-07-24-verifier-asymmetry-check-vs-find.md) — a dozen real tasks
from your repo, scored the same way both times. Prove no regression. While you're
in there, replace few-shot examples in tool prompts with better parameter names
and enums.

**Watch two things.** First, auto-memory. Convenient, but a memory the model
writes about your repo and about you is a log the actor keeps on itself — the same
trust hole [the 07-08 audit-trail dive](./2026-07-08-agent-audit-trail-unattended-commits.md)
flagged for unattended commits. Watch whether it stays legible and correctable or
quietly drifts. Second, watch the altitude per model. "Delete the rules" is
calibrated to the frontier. The cheap open models you route to for the back office
([07-13](./2026-07-13-chinese-models-commodity-tier.md), [07-20](./2026-07-20-cheapest-adequate-model.md))
*are* the dumber model — they still need the scaffolding you're about to strip.
The right prompt is now model-specific, which means a model swap, up or down, is a
portability event. Version your prompts next to your model IDs.

**Ignore two instincts.** Ignore the urge to port your Opus-4.x prompt to Opus 5
unchanged — you'll carry the whole tax and wonder why the new model feels
stubborn. And ignore the overcorrection that reads "less context" and deletes
everything, gotchas and guardrails with the rules. The move isn't less context.
It's *higher-signal* context: fewer rules, the same facts, the same brakes.

The deciding question for every line in your agent's context is small and it
sorts the whole file: *is this teaching the model something it can't infer, or is
it correcting a mistake this model no longer makes?* Keep the first. Delete the
second. The better the model gets, the more of your careful scaffolding turns out
to be the thing holding it back.
