# The Repo Was Never the Lock-In. Watch Who Your Agent Commits As.

*Deep dive · Theo Vance (The Builder) · 2026-08-19 · Cursor built a GitHub alternative; the hard part isn't hosting git, it's who the agent commits as and who reviews it.*

Your agent finishes a feature. It edits six files, runs the tests, commits, pushes a branch, opens a pull request. You read the diff, click merge. Done before lunch.

Now open that commit and read the author line. It says your name. Your email. You did not write a line of it.

That is the small, boring fact under this week's loud one. Cursor shipped [Origin](https://cursor.com/changelog/origin-code-hosting), a code-hosting platform inside the editor — repos, pull requests, merges, and "first serious challenger to GitHub's code-hosting monopoly from an AI-native product company," per the signal that put it at #4 on Hacker News (404 points, 321 comments). The framing is GitHub-killer. The framing is wrong, and it's wrong in a way that tells you exactly what to look at.

## What Origin actually is

Strip the marketing. Origin gives you a Codebase tab, repos at `cursor.com/codebase/<name>`, and a CLI to clone or push. Every repo has pull requests: timeline, commits, checks, changed files, diff comments, merge button. Agents can browse the code, make changes, update PRs, and push branches. Vercel wires in PR preview deploys; Depot and Buildkite run your GitHub Actions workflows. It's in early beta for paid users, and enterprises can opt out.

The load-bearing feature is bidirectional sync. You keep your repo on GitHub. Origin holds a synced copy you browse, search, and pull from; your pushes still go to GitHub, which stays "the source of truth for anything started there." Comment on a PR in Cursor, it posts to GitHub in seconds; react on GitHub, it shows up in Cursor.

Read that as a product engineer and you see the strategy immediately. This is the embrace on-ramp. Keep the incumbent as system of record, make your surface the place people actually work, and wait for the center of gravity to drift. GitLab did the same thing years ago with a GitHub importer; it's the standard way you attack a network you can't beat head-on. Nobody migrates off GitHub on a Tuesday. They just start living somewhere else, and one day the sync direction flips.

## Why "GitHub alternative" is a category error

Here's the part the killer framing misses. **Git is already decentralized.** You have a complete copy of the history on your laptop right now. `git remote add`, push, and you've "migrated" in four seconds. The repository was never the lock-in.

So what does GitHub actually sell? Not storage. It sells the network and the workflow built on top: 180 million developers, 630 million repositories, roughly a billion commits pushed in 2025, and about 43 million pull requests merged every month ([Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)). Plus Actions, plus Issues, plus the identity every other tool trusts when it says "sign in with GitHub." That's the moat — the channel, not the weights, the same shape [we've traced](../2026-W33.md) since the [channel dive](./2026-06-09-channel-was-the-product.md). Fifteen years of challengers — GitLab, Gitea and its Forgejo fork, sourcehut, decentralized Radicle — chipped real pieces off the edges and the graph held anyway, because none of them was a better *git*. Git was never the problem.

Which means Origin's real bet isn't "better hosting." It's two things. One, put the repo where the agent already lives — inside the editor — so the agent reads and writes code without the round-trips GitHub's web API was never designed to serve at machine speed. Two, sync as the migration wedge. Both are reasonable bets. Neither is the interesting one.

## The interesting one is identity

Go back to that commit with your name on it. Ask a plain question: on GitHub today, how does an agent's work get attributed?

Two ways, and both are bad. Either the agent runs with your token, and every commit it makes lands under *your* name — because git's author field is not authenticated. It's a string you set with `git config user.email`, and GitHub simply looks that email up against its user table and renders whoever matches. SSH validates that you're allowed to *push*; it does not validate that the name written into the commit is really you. This isn't obscure — it's why commit spoofing works and why the only thing GitHub will actually vouch for is a [cryptographic signature](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification). "Verified" means the commit was signed with a GPG, SSH, or S/MIME key it can check. Everything else in the author line is self-asserted metadata the server trusts on faith.

Or the agent runs as a GitHub App, and its work shows up under a generic `something[bot]` account — one identity for the whole app, no notion of *which* run, under *whose* authority, against *which* task.

So when a lab tells you [~80% of its merged code is now written by the model](./2026-07-08-agent-audit-trail-unattended-commits.md) (Anthropic's own number about Claude — their claim, not a measured industry figure), the commit log is quietly lying to you. It records either a human who didn't write the code or a bot with no granular identity. There is no first-class, verifiable answer to *which agent produced this change, acting for whom, reviewed by what.*

That is the thing a code host genuinely built for agents would fix. And it's an identity problem, not a hosting one. Origin's own changelog — detailed about sync, PRs, and connected apps — says nothing about agent identity or attribution. The agent "can push branches." As whom?

## And the other half is review

There's a second gap, and it's the one the reader's threads keep circling. A pull request is a human-speed ritual. Someone reads the diff, thinks, comments, approves. When generation runs at machine speed and review runs at human speed, [review becomes the bottleneck and the audited surface](./2026-07-08-agent-audit-trail-unattended-commits.md) — it's the whole reason [serious codebases have started gating AI contributions](../2026-W33.md), not because the code is bad but because the review load is real.

Origin makes authoring, pushing, and merging faster and keeps review in the same PR shape. Read that carefully: making the fast half faster while leaving the slow half untouched *widens* the exact gap. A code host designed for agents wouldn't just speed up the commit. It would change review — agent-reviews-agent gates with a verifiable reviewer identity, checks that a human can trust without reading every line, a record of what was actually looked at. Faster merge is not that. Faster merge is the bottleneck getting worse with a nicer UI.

The tell here is the incumbent looks shakeable — an Ask HN this week had GitHub employees airing internal discontent (208 points; a self-post, so treat it as vibes, not a report). A vulnerable incumbent invites challengers. But you don't beat a network by copying its workflow into a nicer editor. You beat it by fixing the thing the network can't, and the thing GitHub structurally can't fix is that its whole identity and review model was designed when every committer was a person.

## The deciding question

Don't evaluate an "AI-native code host" on stars, on how it looks, or on the word "native." Evaluate it on two axes:

1. **Does it give agents a real, scoped, verifiable identity** — or does it let them keep impersonating you? The commit log should answer "which actor, under whose authority" with a signature, not a spoofable string.
2. **Does it change review to match machine-speed authoring** — or does it only make authoring faster? If it's only the second, it's selling you a bigger bottleneck.

Origin, today, does neither. That's not a verdict against Cursor — it's an early beta, and the sync-first on-ramp is the right opening move. It's a lens for what comes next.

## Do / watch / ignore

**Do.** If you already live in Cursor all day, try Origin as a low-risk read-and-review surface over your existing GitHub repo — sync keeps GitHub as source of truth, so you can walk away with a `git remote` change. And do this regardless of which host you use: turn on commit signing now, and make your agent [sign and co-author its commits](./2026-06-23-git-worktrees-agent-isolation.md). Set a `Co-authored-by:` trailer for the model and sign with your key, so the log records both that you authorized it and that you didn't hand-write it. That's the closest thing to honest attribution the current tooling allows.

**Watch.** Watch for the two things above — a scoped per-agent identity model, and review that changes rather than accelerates. And watch the sync direction. The day pushes default to Origin and GitHub becomes the mirror is the day the wedge became the lock-in. That's the migration completing, and it'll arrive as a quiet settings default, not an announcement.

**Ignore.** Ignore "GitHub killer." Git already made hosting portable; the repo was never what held you. What holds you is Actions, the social graph, and the identity a hundred other tools trust — [the wiring, not the box](./2026-06-09-channel-was-the-product.md). A challenger wins those or it wins nothing, and running your Actions on someone else's copy of your repo isn't winning them. It's renting them back to you.

The commit with your name on code you didn't write is a small thing. It's also the whole story. When the main committer stopped being human, the layer that records who did what should have changed. It hasn't yet. The first host that fixes *that* — not the one with the nicest diff view — is the one worth migrating to.
