# A Subagent Buys You Context, Not Speed

*Claude Code · Kit Sandoval (The Operator) · 2026-07-30 · why the payoff from fanning agents out is the tokens you keep out of your main window — and how to see what the workers you never watched actually did.*

Here is the failure I keep seeing this month. You have a big diff, so you tell
Claude to fan out: one subagent per changed module, review them in parallel, go
faster. A few turns later five one-paragraph summaries land in your
conversation. Four say "no issues." One says "looks good" about a file you
happen to know has a real bug. And you cannot tell whether that subagent read
the file and missed the bug, or never really read it at all — because by default
Claude Code shows you the subagent's tool calls and their results, and *not* its
reasoning. You reached for subagents to move faster, and the thing you actually
bought was five summaries you have to take on faith.

The mistake is upstream of the tooling. It's in what you think a subagent is
*for*.

## The subagent is a context primitive, not a worker pool

Read Anthropic's own framing. You use a subagent "when a side task would flood
your main conversation with search results, logs, or file contents you won't
reference again: the subagent does that work in its own context and returns only
the summary" ([docs](https://code.claude.com/docs/en/sub-agents)). Every
subagent "runs in its own context window." The first bullet on the page is
*preserve context*. Speed is not on the list.

That is the whole game. The value of a subagent is that the 40,000-token test
log, the whole-repo grep, the noisy dependency install — all of it happens in a
window that is not yours, and only the summary comes back. Your main context
stays clean. We've measured why that matters here before: your 200K window has a
[roughly 120K usable speed limit](./2026-06-25-context-budget-sixty-percent.md)
before recall rots, and a loaded MCP config plus the fixed preamble can eat a
[third of the window before you type a word](./2026-07-16-context-tax-before-your-prompt.md).
Against that budget, a subagent that keeps a 40K-token blob out of the main
thread is worth about a third of your working memory. That is the leverage.
Wall-clock savings are a side effect, and when you chase the side effect you get
the blind fan-out above.

This is the sharper version of a point [June made in June](./2026-06-13-subagent-fan-out-budget.md):
every nested subagent spins up a *fresh* context you pay for, so fan-out has a
token bill. True. But the reason to sign that bill isn't parallelism — it's
isolation. Ask one question before you spawn: *does this task produce output I'd
have to scroll past forever?* If yes, isolate it. If no, a subagent is just
overhead with a summary attached. The deciding quantity is **tokens kept out of
your main window**, not seconds saved.

## The two questions Anthropic just answered

The Claude Code changelog spent July settling the two real questions of running
a tree of these things: how deep it goes, and how much of it you can see.

**How deep.** A subagent can spawn its own subagents, and as of v2.1.217 the
default is "up to three layers below the main conversation," set with
[`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`](https://code.claude.com/docs/en/sub-agents)
— `1` turns nesting off entirely. (On v2.1.172 through v2.1.216 it was five
layers and you couldn't change it; the changelog chatter about "depth 1 → 3" is
really *the limit became yours to set*.) Depth is not free real estate. The
shape that earns it is narrow and specific — the docs name it: "a reviewer
subagent that dispatches a verifier per finding, so the intermediate output
never reaches your main conversation. Only the top-level subagent's summary
returns to you."

That pattern is not hypothetical. It is exactly what Anthropic shipped as managed
[Code Review](https://code.claude.com/docs/en/code-review): "a fleet of
specialized agents examine the code changes… Each agent looks for a different
class of issue, then a verification step checks candidates against actual code
behavior to filter out false positives. The results are deduplicated, ranked by
severity." Find, then verify each candidate, then dedup — a whole layer of
noise dying inside the tree instead of in your window. It works because it's
search against a cheap checker, which is
[the one shape agents reliably win at](./2026-07-24-verifier-asymmetry-check-vs-find.md).
Nest for that. Don't nest three deep because you can.

**How visible.** This is the fix for the blind fan-out, and it's one flag. By
default Claude Code "emits only subagent `tool_use` and `tool_result` blocks."
Pass [`--forward-subagent-text`](https://code.claude.com/docs/en/headless) (or
set `CLAUDE_CODE_FORWARD_SUBAGENT_TEXT`), added in v2.1.211, and it also emits
each subagent's text and thinking "so you can reconstruct each subagent's
transcript." Better: every forwarded message carries a `parent_tool_use_id`
pointing at the Agent call that spawned it, so you can rebuild the entire nesting
tree by following those IDs — and as of v2.1.219 that finally includes messages
from *nested* subagents, which simply didn't appear in the stream before. The
catch is that the flag only does anything with `--print` and
`--output-format stream-json`; it's a headless/observability switch, not an
interactive one.

So the honest rule: if you are going to trust a summary from an agent you didn't
watch, forward its text. Otherwise "looks good" is a claim with no receipt.

## Make the return checkable, not chatty

The other half of trusting a summary is shaping it. A subagent's final message
*is* its return value — that's it, the whole interface. If you let that be a
paragraph of prose, you've built a system whose contract is vibes. In headless
runs, close the gap: `--output-format json` with
[`--json-schema`](https://code.claude.com/docs/en/cli-reference) forces the
result to match a schema you define, landing the structured object in a
`structured_output` field you can gate a build on. "Return a JSON list of
`{file, line, severity, evidence}`" is a contract. "Review this module" is a
mood. The same verifier discipline from last week's dive applies to your own
orchestration: [write the check first](./2026-07-24-verifier-asymmetry-check-vs-find.md),
even when the thing being checked is your subagent's report.

## The guardrails that still apply

Three things that bite specifically because subagents now run in the background
by default (since v2.1.198):

- **Parallel writers still corrupt a shared checkout.** If two subagents edit
  files at once, give each its own with `isolation: worktree` in the frontmatter
  — the same [worktree-per-agent isolation](./2026-06-23-git-worktrees-agent-isolation.md)
  Bun's fleet leaned on, now a one-line field.
- **Results arrive late, and failures no longer masquerade as findings.** A
  background subagent's result "reaches Claude as a completion notification in a
  later turn"; ask about progress early and it'll tell you the agent's still
  running. And since v2.1.199, a subagent killed by a rate limit or a server
  error reports *that* rather than returning the error text as if it were the
  answer — which is the difference between "no issues found" and "the run died"
  on a day like [the 41-minute API outage on July 29](https://status.claude.com/incidents/q2kg8n613kr3).
  Don't let a dead agent read as a clean one.
- **The reviewer went off-camera too.** `/code-review` now runs as a background
  subagent "with its own context window, so it doesn't fill your conversation"
  (v2.1.218). Good for your context; it also means the review happens where you
  can't see it. That's the [deskilled-reviewer trap](./2026-07-22-human-in-the-loop-deskilled.md)
  wearing new clothes — the brake you can't watch is a brake you can't trust. If
  it matters, forward its text and read what it actually checked.

## Do this before lunch

Set the tree's shape once, in `settings.json` — cap depth to what your workflows
actually use, and raise the concurrency ceiling only if you're deliberately
running wide (the default blocks a 21st concurrent subagent):

```json
{
  "env": {
    "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "2",
    "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "20"
  }
}
```

When you fan out and want to *see* the tree, run headless with text forwarding
and follow the transcript live ([headless docs](https://code.claude.com/docs/en/headless)):

```bash
claude -p "Review each changed module; return findings only" \
  --output-format stream-json --verbose --forward-subagent-text \
  | jq -c 'select(.parent_tool_use_id != null) | {agent: .parent_tool_use_id, msg: .message.content}'
```

And make the workers return data, not paragraphs — put the contract in the
subagent file (`.claude/agents/reviewer.md`):

```markdown
---
name: reviewer
description: Reviews one module for correctness bugs. Use per changed module.
tools: Read, Grep, Glob, Bash
model: sonnet
isolation: worktree
---
Review the assigned module for correctness bugs only. Return a JSON array of
{file, line, severity, evidence} — evidence must be a file:line citation, not
an inference from naming. Return [] if you find nothing.
```

**Watch:** whether text forwarding ever becomes the default. Today it's opt-in
for a reason — the lean stream is what keeps subagents cheap on context. My call:
through Q1 2027 it stays opt-in, the default single-session concurrent limit stays
20 and default depth stays 3, and Anthropic keeps pushing *sustained* parallelism
to [agent teams](https://code.claude.com/docs/en/agent-teams) and background
sessions — each with its own context — rather than scaling up the single-context
subagent. Confidence 65%. If the default flips, or the concurrency ceiling jumps,
that's Anthropic repositioning subagents as a compute primitive instead of a
context one, and this whole piece's advice inverts.

**Ignore:** the urge to go three deep because the setting allows it. Nesting pays
in exactly one case — a fan-out whose intermediate output you never want to see,
like find-then-verify. Everywhere else, a flat set of well-scoped subagents,
each returning a schema you can check, beats a deep tree you're trusting on
faith. The agent supplies the parallelism. You supply the isolation boundary and
the contract — and those, not the layer count, are the ceiling on everything the
tree hands back.
