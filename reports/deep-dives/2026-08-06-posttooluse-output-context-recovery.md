# Your Context Window Is 60% Tool Output. A PostToolUse Hook Takes It Back.

*Claude Code · Kit Sandoval (The Operator) · 2026-08-06 · why the biggest drain on your context budget isn't the preamble — it's what your tools print back, and how to compress it before the model reads it.*

Here's the failure. Your agent is sharp for the first thirty turns. By turn forty it's slower, repeats itself, and forgets a decision you watched it make. You reach for the usual suspects — a bloated system prompt, too many MCP servers, a model having a bad day.

Run `/context` and look at what's actually eating the window. It's not the preamble. It's `git status`.

I spent a month blaming the [fixed preamble](../deep-dives/2026-07-16-context-tax-before-your-prompt.md) — the ~33k tokens Claude Code sends before your first keystroke. That's real, and worth trimming. But it's paid **once**. The thing that grows all session, uncapped, is the output of your tools. Every `npm test`, every `git diff`, every `curl` dumps its full result into the conversation and leaves it there for the rest of the run.

An audit posted to the Claude Code tracker ([issue #32105](https://github.com/anthropics/claude-code/issues/32105)) put a number on it: across 8 sessions, 603 tool calls, 626K tokens, **tool results were ~60% of context tokens**. Every session was over 49%. The worst hit 73.6%. Their estimate: ~82% of those tokens were compactable — noise the model never needed to reason over. Treat the exact percentages as one team's measurement, single-sourced. But the direction is checkable on your own repo in five minutes, and it will not surprise you.

## Input doesn't predict output

Two months ago I wrote that [hooks are the real guardrail](../deep-dives/2026-07-02-hooks-are-the-real-guardrail.md) — that a `PreToolUse` hook, real code reading `.tool_input.command`, is the only thing that reliably stops a command from running. That still holds. But it's the wrong tool for this problem.

`PreToolUse` gates what goes **in**. For Bash, input tells you almost nothing about output size. The same `git status` returns 5 tokens on a clean repo and, per that audit, **5,491 tokens** on one with 200+ untracked files — a 1,098× spread. `git diff`: 20 to 6,211. `docker logs`: 10 to 5,000. You cannot know from the command string whether the result is 5 tokens or five thousand. As the issue puts it: "the only clean solution is post-execution result modification."

That's what shipped. The changelog line is one sentence: *"PostToolUse hooks can now replace tool output for all tools via `hookSpecificOutput.updatedToolOutput` (previously MCP-only)."* The capability existed for MCP tools; the recent [change generalized it to every built-in tool](https://www.agentpatterns.ai/tool-engineering/posttooluse-output-replacement/) — Bash, Read, Edit, the lot. This week's scout flagged it, which is why it's today's edition.

The mechanism, from the [hooks reference](https://code.claude.com/docs/en/hooks): a `PostToolUse` hook fires after a tool runs and receives the full result on stdin —

```json
{
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input":  { "command": "git status" },
  "tool_response": { "stdout": "…5,491 tokens of untracked files…" },
  "tool_use_id": "toolu_01ABC…"
}
```

— and returns a replacement:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "updatedToolOutput": { "stdout": "…200-token summary…" }
  }
}
```

The model receives `updatedToolOutput` **instead of** the original. The tool already ran; you're not changing what happened, only what the model reads about it. On the `git status` example the issue reports the hook compressing 5,491 tokens to ~200 — a 94% per-call cut, and across a 257-call session, ~35% of the total context budget recovered.

## Why this is the brake that was missing

Stack it against what we already knew. The usable window is [~60% of the advertised 200K](../deep-dives/2026-06-25-context-budget-sixty-percent.md) — call it 120K before accuracy falls off. Now put ~60% of everything you spend into raw tool output, most of it compactable. You are filling a shrunk window with noise, then wondering why the model degrades.

And you cannot cache your way out. Prompt caching refunds **dollars** on a byte-identical prefix; it does not refund the **window** or the **attention** — the same point that made the preamble worth trimming even though it caches perfectly. A tool result that lands in the transcript occupies the window for the rest of the session whether or not you paid for it twice. Compression at write-time is the only move that reclaims the two things caching can't.

There's a second brake on the output side, and you should know when to use which. A [subagent keeps verbose work off the main window entirely](../deep-dives/2026-07-30-subagents-context-not-speed.md) — run the noisy test suite in a subagent and only its conclusion returns. That's the right tool when a whole *task* is noisy and self-contained. `updatedToolOutput` is the surgical version: you want the command in the main loop, you just don't want its 6,000-line log. Subagent for the task; hook for the call.

## The catch, honestly

This is a sharp instrument, and it cuts toward you too.

**The output you replace is the output that gets recorded.** The modified result is what's written to the transcript; the original isn't preserved anywhere Claude can reach. That's the point for secrets — but it means a careless compressor can hide a failure from your own [audit trail](../deep-dives/2026-07-08-agent-audit-trail-unattended-commits.md). Telemetry helps: OpenTelemetry tool spans capture the *original* output before the hook runs, so the untouched result still exists off to the side. Don't lean on the transcript alone as your record once a compression hook is live.

**The tool already ran.** `PostToolUse` can't un-`rm` a file or un-send a request. Compression is not containment — that's still the [sandbox and PreToolUse](../deep-dives/2026-07-02-hooks-are-the-real-guardrail.md)'s job. This only governs what the model *sees*.

**Read the right field.** A real bug report ([agentmemory #539](https://github.com/rohitg00/agentmemory/issues/539), single-sourced) traced a compression hook silently failing because it read `data.tool_output` while Claude Code sends `data.tool_response` — the shape mismatch cost ~47% of the output. The field is `tool_response`, and its text lives in `stdout` for Bash. Reflect the shape you received back into `updatedToolOutput`; don't hand-build a schema and hope.

**Don't compress what you'll need.** Head-and-tail beats a summarizer here for the same reason [dump-to-markdown beats /compact](../deep-dives/2026-06-25-context-budget-sixty-percent.md): you decide what survives, deterministically, instead of trusting a model to summarize under load. Keep the first lines and the last lines — the last are usually the error — and leave the agent a note on how to get the middle back.

## Paste this in before lunch

A `PostToolUse` hook that compresses any Bash output over ~2,000 tokens to head+tail, and tells the agent how to recover the rest. `.claude/hooks/squeeze.py`:

```python
#!/usr/bin/env python3
# Compress verbose Bash output before it lands in the context window.
import json, sys

data = json.load(sys.stdin)
resp = data.get("tool_response", {})        # tool_response, NOT tool_output (issue #539)
text = resp.get("stdout") or resp.get("output") or ""
LIMIT = 8000                                # ~2,000 tokens; tune per repo

if len(text) <= LIMIT:
    sys.exit(0)                             # small output: leave it untouched

lines = text.splitlines()
kept  = "\n".join(lines[:40])
tail  = "\n".join(lines[-20:])
squeezed = (f"{kept}\n"
            f"… [{len(lines)} lines / {len(text)} chars compressed by squeeze hook] …\n"
            f"{tail}")

out = dict(resp)
out["stdout" if "stdout" in resp else "output"] = squeezed

print(json.dumps({
    "hookSpecificOutput": {
        "hookEventName": "PostToolUse",
        "updatedToolOutput": out,
        "additionalContext": "Bash output was compressed to head+tail. "
                             "Re-run with grep/tail/--quiet if you need the middle."
    }
}))
```

Wire it in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Bash",
        "hooks": [
          { "type": "command",
            "command": "python3 $CLAUDE_PROJECT_DIR/.claude/hooks/squeeze.py" } ] }
    ]
  }
}
```

- **Do** run a session first *without* the hook, check `/context`, and see your own tool-result ratio before you tune `LIMIT`. The 60% figure is one team's; yours is a five-minute measurement.
- **Do** add a redaction hook on the same event if you touch cloud creds — the [community redaction hooks](https://github.com/l-mb/claude-code-redaction-hooks) `sed` AWS keys and bearer tokens out of `tool_response` before the model can echo them into a commit message. Redaction is the one case where *losing* the original is the goal.
- **Watch** the audit gap: once compression is live, trust OTel spans or git history for the real record, not the transcript.
- **Ignore** the reflex to summarize with a model. Deterministic head+tail is cheaper, faster, and it can't hallucinate a passing test into a log that failed.

The window didn't get smaller this year. The frontier keeps advertising a million tokens. But the [usable](../deep-dives/2026-06-25-context-budget-sixty-percent.md) fraction is stubborn, and until now the largest single consumer of it — what your own tools print back — was the one input you had no handle on. Now you do. It's one file and one matcher. `git status` doesn't get to spend 5,000 tokens telling you about files you didn't ask about.
