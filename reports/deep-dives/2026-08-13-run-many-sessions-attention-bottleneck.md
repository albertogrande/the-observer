# You Can Run Fifteen Claudes. The One That Can't Scale Is You.

*Claude Code · Kit Sandoval (The Operator) · 2026-08-13 · the session — not the subagent — is Claude Code's real unit of parallelism, and the control plane for it shipped this week*

Here is the failure. You read that the way to go fast is to run more than one
Claude, so you open eight terminal tabs, kick off eight tasks, and start
cycling. Tab 1 is waiting on a permission prompt. Tab 4 finished six minutes
ago and has been sitting idle. Tab 6 asked you a clarifying question you never
saw. Tab 3 is quietly wrong and you won't find out until you get to it. You are
not running eight agents. You are hand-scheduling eight agents with your
eyeballs, and your eyeballs are the slowest part of the system.

I've spent this desk's last two months on the *inside* of one session — the
120K usable window ([Jun 25](./2026-06-25-context-budget-sixty-percent.md)),
the 33K preamble tax ([Jul 16](./2026-07-16-context-tax-before-your-prompt.md)),
the tool output that eats 60% of the budget
([Aug 6](./2026-08-06-posttooluse-output-context-recovery.md)). This week the
constraint moved. It's no longer the window. It's the human trying to keep
fifteen windows in his head at once.

## The subagent was never the concurrency primitive

Start by killing a category error. Two weeks ago I argued that a Claude Code
subagent is a **context-isolation** primitive, not a speed one — you fan out to
keep a 40K-token log out of your main window, and any wall-clock win is a side
effect ([Jul 30](./2026-07-30-subagents-context-not-speed.md)). That's
still true, and Anthropic's own docs now say it plainly: a subagent has its
"own context window; results return to the caller," and it can "report results
back to the main agent only"
([agent teams doc](https://code.claude.com/docs/en/agent-teams)). A subagent
lives and dies inside one parent session. Close the parent and it's gone. It is
a way to spend context, not a way to run work in parallel that survives you.

The thing that actually runs in parallel — the unit that keeps working when
your terminal closes, that lives on another machine, that you can address by
name — is the **session**. And this week Anthropic shipped the missing pieces
that turn a pile of sessions into a fleet you can drive.

Three of them, from the [changelog](https://code.claude.com/docs/en/changelog):

- **Cross-session messaging** (v2.1.224): "Claude Code sessions can now message
  each other, on any of your machines, with `ListAgents` to discover them." A
  session is now an *addressable* thing. A driver session can list its workers
  and hand them tasks.
- **Teleport** (v2.1.223): `claude --teleport <session id>` continues a cloud
  session on your local machine — start a task from your phone or the web,
  finish it in your terminal.
- **The self-hosted runner** (v2.1.225): `claude self-hosted-runner` "turns your
  own machines or containers into a place Claude Code web, mobile, and desktop
  sessions can run." Your laptop is no longer the only executor.

Put those next to background sessions — parked with `/background` or `←`,
surfaced in the `claude agents` view — and you have the shape of a scheduler:
independent workers, running anywhere, addressable, backgroundable, durable.

This isn't a thought experiment. Boris Cherny, who built Claude Code, runs it
this way: roughly **5 terminal sessions across 5 separate git checkouts**, tabs
numbered 1–5, plus **5–10 more on claude.ai** from the browser and phone —
ten-to-fifteen Claudes at once, handed between local and cloud
([VentureBeat](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are),
[xda](https://www.xda-developers.com/set-up-claude-code-like-boris-cherny/);
the exact counts are one person's practice, secondhand, so treat them as the
shape, not a spec). Notice what he is *not* running: not one session with
fifteen subagents. Fifteen sessions.

## The bottleneck you just moved onto yourself

Once the sessions are independent, the scarce resource is your attention. And
attention doesn't parallelize. You can only look at one transcript at a time,
and the cost of being wrong about *which* one to look at is a session sitting
blocked while you read a different one.

The fix Anthropic shipped is a control plane plus a routing primitive.

The control plane is `claude agents`: a board that groups every session by
**needs input / working / completed** instead of making you tab through them to
find out. As of v2.1.212 it even shows a session waiting on a sandbox or MCP
prompt as "Needs input" rather than a misleading "Working." Stop polling tabs;
watch one board.

The routing primitive is the one most people miss, because it shipped as a
one-line changelog entry back in v2.1.198: background sessions "that need input
or finish now fire the `Notification` hook (`agent_needs_input` /
`agent_completed`)." That is the whole game. It means you can make your
computer *tell you* which session needs you, instead of going to look. The
[hooks doc](https://code.claude.com/docs/en/hooks) lists the matchers you care
about on the `Notification` event: `permission_prompt`, `idle_prompt`,
`agent_needs_input`, `agent_completed`. The hook receives `session_id`, `cwd`,
and `notification_type` on stdin.

Here is the paste-in. Put this in `~/.claude/settings.json` and every session
you run — foreground or background — pings you the moment it blocks or
finishes, with the working directory so you know *which* one:

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "agent_needs_input|permission_prompt|idle_prompt",
        "hooks": [
          { "type": "command",
            "command": "d=$(jq -r '.cwd'); osascript -e \"display notification \\\"needs you: $d\\\" with title \\\"Claude\\\" sound name \\\"Ping\\\"\"" }
        ]
      },
      {
        "matcher": "agent_completed",
        "hooks": [
          { "type": "command",
            "command": "d=$(jq -r '.cwd'); osascript -e \"display notification \\\"done: $d\\\" with title \\\"Claude\\\"\"" }
        ]
      }
    ]
  }
}
```

The matcher is a regex, so the first entry catches all three "I'm blocked"
notification types at once. The hook reads the event JSON on stdin — I pull
`.cwd` with `jq` so the banner names *which* session — and on Linux you swap the
two `osascript` calls for `notify-send Claude "needs you: $d"`. The point isn't
the shell: `Notification` output is discarded except for terminal sequences, so
a side-effecting command (beep, banner, or a `curl` to your phone's push
service) is exactly what the event is for. Your attention now gets *pulled* to the one
session that's blocked, instead of you round-robining fifteen of them hoping to
catch the blocked one before it's cost you ten minutes.

## Don't confuse the three ways to run more than one Claude

The mistake I keep seeing is reaching for **agent teams** when you want
**separate sessions**, or vice versa. They are not the same tool, and the docs
are unusually blunt about it. Three tiers, one deciding question:

| | Subagent | Agent team | Separate sessions |
|---|---|---|---|
| Scope | inside one session | inside one lead session | fully independent |
| Talk to each other? | no (reports to parent) | yes (shared task list + mailbox) | yes (cross-session `SendMessage`) |
| Survives a restart? | no | no (`/resume` doesn't restore teammates) | yes |
| Runs on another machine? | no | no | yes |
| Token cost | lowest | highest (each is a full instance) | one full instance each |

Agent teams are the collaborative middle: a lead spawns teammates that share a
task list and message each other through a mailbox at
`~/.claude/teams/{team}/inboxes/`. They're genuinely good for a parallel review
or competing-hypotheses debug where the workers must *challenge each other*. But
they're **experimental** (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, off by
default), there's **one team per session**, **no nested teams**, and **no
session resumption** — resume and the lead starts messaging teammates that no
longer exist. That's a collaboration tool, not a production concurrency model.

The deciding question: **do your workers need to talk to each other
mid-flight?** If yes — the security reviewer needs to hand a finding to the perf
reviewer — use a team. If no — five independent modernization tasks that each
just need to get done and survive your laptop lid closing — use separate
sessions. Boris runs separate sessions because his tasks are independent, he
wants them durable, and he wants to move them to the cloud. That's the tell for
most real work.

## The catches, because there always are some

More sessions is not more throughput past your attention budget. The team doc's
own guidance — "start with 3–5," "diminishing returns beyond a certain point" —
applies to *you*, the coordinator, not just to token cost. You moved the
bottleneck onto yourself; it still has a ceiling. The honest metric isn't
sessions-running, it's **things-shipped per unit of your attention**, and past
some N the notification hook is just interrupting you faster than you can act.

Three more, quickly. Every session pays the full fixed cost — the ~33K preamble
([Jul 16](./2026-07-16-context-tax-before-your-prompt.md)) times N, plus a git
checkout times N on disk; parallelism multiplies the startup tax you spent all
summer trying to cut. Background sessions now commit and push on their own
(v2.1.221/224), which means writing goes off-camera again — the `claude agents`
board shows you a green "completed" badge, not a diff, so trust the git log, not
the badge ([Jul 8](./2026-07-08-agent-audit-trail-unattended-commits.md)). And
if your workers write in parallel, you still need isolation — separate checkouts
(Boris's blunt version) or `isolation: worktree`
([Jun 23](./2026-06-23-git-worktrees-agent-isolation.md)), a guard that leaked
into the shared checkout enough times this summer that the changelog reads like
a patch diary (v2.1.216, .222, .223, .224). One upside worth naming: a message
from another session is treated as **untrusted** — it can't approve a permission
prompt on your behalf — so a driver session can delegate work but can't click
through your guardrails. That's the right default, and it's also the reason full
hands-off orchestration isn't here yet.

## Before lunch

Do three things. **One:** paste the `Notification` hook above into
`~/.claude/settings.json` and stop tabbing to check status. **Two:** run your
next batch as separate backgrounded sessions (`/background` or `←`) and drive
them from the `claude agents` board, not from fifteen terminal tabs. **Three:**
reach for an agent team *only* when your workers must talk to each other, and
keep it to 3–5.

The subagent buys you context. The team buys you collaboration. The session
buys you parallelism — and this week it finally got a control plane and a
doorbell. Watch for one signal to know whether this becomes the default way to
work: a `Notification`-hook-plus-`claude agents` fan-out pattern showing up in
people's dotfiles the way the 60% context rule did. If the fleet stays
something only the tool's creator can hold in his head, the primitives shipped
too early. My bet is they didn't.
