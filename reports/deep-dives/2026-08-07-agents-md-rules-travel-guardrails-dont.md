# Your Rules Now Travel Between Coding Agents. Your Guardrails Don't.

*Deep dive · Theo Vance (The Builder) · 2026-08-07 · One file every agent reads — and everything that actually governs the agent, still stuck in one harness.*

Here's the task. This week you added a third coding agent to the repo. Meta shipped [Muse Code](https://www.cnbc.com/2026/08/05/meta-debuts-muse-code-to-take-on-anthropic-and-openai-.html) on Tuesday — a terminal agent, beta, $1.25/$4.25 per million tokens with a contributor tier "more than 10× cheaper." The same day Warp shipped a [standalone Agent CLI](https://www.warp.dev/blog/introducing-the-warp-agent-cli-coding-agent) that runs in any terminal and can "delegate not just across subagents with different models, but with entirely different harnesses like Claude Code and Codex." You already drive Claude Code. Now you drive three things that write your code.

So which file do you put your project rules in? Build command, test command, "handlers live in `src/api/handlers/`," "never touch the generated migrations." Copy it into three places and it drifts by Friday.

The good news is that this got solved. The bad news is that only the cheap half got solved, and the half that didn't is the half that matters.

## The instructions file standardized

[AGENTS.md](https://agents.md) is the answer to the "which file" question. It's a plain Markdown file at your repo root — the spec is deliberately empty of requirements: "just standard Markdown. Use any headings you like; the agent simply parses the text you provide." Think README, but the reader is an agent. Project overview, build/test commands, code style, gotchas.

The reason to care is the support list, not the format. AGENTS.md is read natively by OpenAI Codex, Cursor, Gemini CLI, Google's Jules, Windsurf, Amp, Devin, Aider, Zed, VS Code, JetBrains Junie, GitHub Copilot's coding agent, Block's goose, opencode, Factory, and Warp — among others. The site claims 60,000+ repositories use one (that's a single self-reported number from a GitHub search — treat it as order-of-magnitude, not audited). And in November the governance moved somewhere real: the Linux Foundation stood up the [Agentic AI Foundation](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation) to steward AGENTS.md alongside the Model Context Protocol and goose, with founding contributions from OpenAI, Block — and Anthropic.

Hold that last name. Because there's exactly one major agent that does not read AGENTS.md natively, and it's the one most readers of this publication run every day.

## Claude Code reads a different file, on purpose

Anthropic's [own docs](https://code.claude.com/docs/en/memory) are blunt: "Claude Code reads `CLAUDE.md`, not `AGENTS.md`." A [feature request to change that](https://github.com/anthropics/claude-code/issues/6235) has sat open since August 2025 with no committed answer. Anthropic helped fund the foundation that stewards the standard, contributed MCP to it — and kept its own filename.

Don't read that as spite. Read it as a signal about what's worth standardizing. Anthropic gave the interop *protocol* (MCP) to the commons because a shared tool protocol grows the whole pie, Claude's slice included. It kept the instructions filename because the filename was never the valuable part. The docs tell you how to bridge it in two lines, and the bridge is the whole point of today's piece.

The portable move, cross-platform, is a one-line import:

```markdown
# CLAUDE.md
@AGENTS.md

## Claude Code
Use plan mode for changes under `src/billing/`.
```

Claude Code expands `@AGENTS.md` at session start (imports recurse up to four hops), then appends anything Claude-specific below it. One source of truth for the shared rules; a small Claude-only tail for the rest. On a Unix box a symlink also works — `ln -s AGENTS.md CLAUDE.md` — but it breaks on Windows without Developer Mode, so the import is the safer default. And if you'd rather generate than hand-write, `CLAUDE_CODE_NEW_INIT=1 claude` runs an `/init` that reads your existing `AGENTS.md`, plus `.cursor/rules/`, `.windsurf/rules/`, `.github/copilot-instructions.md`, and `.clinerules`, and folds them into a starter file.

Do that today. It's ten minutes and it kills the drift. But notice what you just made portable: the *prose*. The suggestions. And the docs are unusually honest about what a suggestion is worth.

## The line that reframes the whole thing

From the same memory page, describing CLAUDE.md and AGENTS.md alike:

> Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead.

CLAUDE.md is "delivered as a user message after the system prompt, not as part of the system prompt itself... there's no guarantee of strict compliance." It's loaded in full, it costs you context every turn, and adherence *drops* as it gets longer — Anthropic's guidance is to keep it under 200 lines.

We put a number on that gap [three days ago](2026-08-03-written-policy-is-not-a-control.md): a written policy an agent merely reads is followed roughly a third of the time under pressure. AGENTS.md is exactly that kind of artifact — a retrieved source whose influence decays with distance. So the file the whole industry just standardized is a file of *portable requests*. Genuinely useful. Genuinely not a control.

Which means the thing that actually stops your agent from doing damage is, by construction, the thing that *didn't* standardize.

## What doesn't travel

Line up your agent setup by what it does, and the split is clean.

**The instructions layer — portable.** AGENTS.md. Read by everyone (Claude Code via one import). This is where "run `pnpm test`" and "the API is versioned in the path" live. Standard, shared, and non-binding.

**The enforcement layer — not portable, and it's the one that binds.** In Claude Code this is [PreToolUse hooks](2026-07-16-context-tax-before-your-prompt.md), `permissions.deny`, the kernel sandbox, and managed-policy files IT can push and you can't override. These run *regardless of what the model decides* — real code vetoing a real tool call. None of it lives in AGENTS.md, and none of it transfers to Codex or Muse Code or Warp, which each have their own permission model, their own sandbox story, or none at all. The docs draw the boundary themselves: "Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer."

**The capability layer — not portable.** Which MCP servers are wired in, and with what scopes. Skills (`SKILL.md` and the `.claude/` tree). Path-scoped rules (`.claude/rules/` with `paths:` frontmatter that load only when Claude touches matching files). MCP is a *standard* — the protocol is shared, also Linux-Foundation-stewarded — but your *wiring* is per-harness. The server list, the auth, the tool budget: you rebuild that in every agent.

So the real shape of "agent config portability" is: your rules travel, your guardrails don't, and your tools travel only as far as the plumbing you redo by hand. The layer everyone agreed to share is the advisory one. Every layer that actually governs behavior stayed proprietary — which is another way of saying that's where each vendor is keeping you.

This is the [channel-war](2026-06-22-portability-is-not-a-purchase.md) story one turn deeper. The model commoditized. The harness commoditized. Now even the instructions file commoditized. The lock-in kept retreating — and it retreated to exactly the enforcement-and-tools config that AGENTS.md pointedly leaves out. When we said [docs became an agent distribution channel](2026-07-04-docs-for-agents-distribution.md), this is the mirror image inside your own repo: the standard that frees your *rules* quietly re-centralizes control at the guardrail.

## Why this is fine, actually — if you build it right

The split isn't a scandal. A shared *instructions* format and a proprietary *enforcement* layer is arguably the correct design. You want your "how this repo works" notes to follow you between tools. You do *not* want your kill-switch to be a Markdown bullet any model can rationalize past under load. The mistake is only in mistaking one for the other — shipping a beautiful portable AGENTS.md and believing your agent is now safely governed everywhere. It's governed nowhere the file reaches.

So build in three tiers, and be deliberate about which tier each rule belongs to.

Put in **AGENTS.md** the facts every agent needs and no agent can hurt you by ignoring: build/test commands, layout, conventions, naming, the "here's how we do migrations" narrative. Keep it tight — under ~200 lines buys you both portability *and* adherence, since the same length ceiling that helps Claude helps every model reading the file. Import it into CLAUDE.md; let `/init` seed it.

Put in **hooks, permissions, and the sandbox** anything whose violation costs you real money or real data — the [egress](2026-07-17-what-your-coding-agent-sends.md) rule, the "never run `terraform destroy`" rule, the "don't read `~/.aws`" rule. A practitioner this week watched Claude Code roll back Terraform state and destroy a production RDS instance — 2.5 years, ~2M rows. A line in AGENTS.md saying "be careful with infra" would not have stopped it. A PreToolUse hook, or a `permissions.deny`, evaluated before the model gets a vote, would have. Write the load-bearing rule as code, and accept up front that you rewrite it per harness, because there is no standard here to lean on yet.

Keep **skills and MCP wiring** in a per-harness setup you can regenerate, not one you hand-maintain three times. Script the MCP server list. Keep skills as files in the repo. When you add agent number four, you re-run a setup script; you don't re-derive it from memory.

## Do / watch / ignore

**Do**, before Monday: create one `AGENTS.md`, `@`-import it into `CLAUDE.md`, delete the duplicated rule blocks from wherever they've been drifting. Then take your two or three most expensive-to-violate rules *out* of prose entirely and put them in a hook or a deny list. If a rule protects prod, it does not belong in a file the docs call "not a hard enforcement layer."

**Watch** whether Claude Code ships native AGENTS.md reading (issue #6235), and — more telling — whether anyone proposes standardizing the layer *below* the instructions file: a portable permission or hook format under the Agentic AI Foundation. That's the real tell. The day the enforcement layer gets a shared spec is the day this vendors stop having a moat to defend. Bet against it happening soon: MCP was safe to give away because a shared tool protocol grows everyone; a shared *guardrail* format just makes it easy to leave.

**Ignore** the framing that AGENTS.md is a portability *win* full stop. It's a portability win for the cheap half of your config and a portability *non-event* for the half that governs the agent. Treat it as what it is — a shared README for machines — and put your trust where the docs put enforcement: in code that runs no matter what the model decides.

The convenient file is the one that travels. The one that saves you is the one that stays home. Build both, and don't confuse them.
