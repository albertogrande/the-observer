# Your `.gitignore` Fails Open. Agents Are the Reason to Flip It.

*Deep dive · Theo Vance (The Builder) · 2026-09-06 · The default git tracks everything and subtracts exceptions — that model was written for humans who create files on purpose.*

You let an agent run for an hour. You came back to a green checkmark and fourteen files you never asked for. A scratch `CLAUDE.md` it wrote to think out loud. A `coverage.out` from the test run. A `.env.local` it created to hold a throwaway key while it debugged an integration. A `screenshot-2.png` from a browser step. Two `tmp/` directories. You skim the diff, it looks right, you type the reflex:

```bash
git add .
git commit -m "wire up the webhook"
```

Now answer one question without looking: what did you just stage?

The honest answer is that you don't know. You know what you *meant* to stage. The gap between those two is where secrets get published, and this week [GitGuardian put a number on the gap](https://blog.gitguardian.com/the-state-of-secrets-sprawl-2026/): 28.65 million new hardcoded secrets hit public GitHub in 2025, up 34% year over year — the biggest single-year jump they've recorded. The part that should stop you is the breakdown. GitGuardian measured Claude Code–assisted commits leaking secrets at **3.2%**, versus a **1.5%** baseline across all public commits. The agent-assisted commit publishes a live credential more than twice as often as the average one.

That is not because the agent is careless. It's because git's default is a **denylist**, and the denylist fails open.

## The default was written for a slower author

Here's the model you're actually running. Git tracks everything in the tree by default. Your `.gitignore` is a list of *subtractions* — named exceptions you've learned to remove. `node_modules/`. `.DS_Store`. `*.log`. Every entry is a scar from a time something junk got committed and someone added a line so it wouldn't happen again.

That model has a specific failure mode: **anything you did not anticipate is committed by default.** A denylist is only as good as your ability to predict, ahead of time, every category of file that will ever appear in the tree. For twenty years that was a fine bet, because the author was a human typing `touch` on purpose. You knew roughly what files your project made.

The [blog post that put this on the front page this week](https://packagemain.tech/p/gitignore-everything-by-default) — Alex Pliutau's, discussed [hard on HN](https://news.ycombinator.com/item?id=49576258) — opens on a 207-line `.gitignore` and a pile of "various agentic docs or subfolders" nobody meant to keep. That's the tell. The `.gitignore` grows without bound now, and it still loses, because the thing generating novel file types is no longer you. It's a process that writes to your working tree unattended, at machine speed, inventing filenames you've never seen. You cannot enumerate ahead of time what an agent will create. The denylist's core assumption just broke.

## Flip it: ignore everything, name what you keep

The inversion is one file. Ignore `*`, then un-ignore the handful of things you actually track:

```gitignore
# ignore everything
*

# then allow, explicitly
!.gitignore
!*.go
!go.mod
!go.sum
!README.md
!/cmd/
!/internal/
```

Now the default is reversed. A file the agent invents — `CLAUDE.md`, `scratch.py`, `.env.local`, `debug-3.png` — matches `*`, matches nothing on the allow side, and stays out of `git add .` by construction. It **fails closed.** To commit a genuinely new kind of file, you have to name it. That is a deliberate act, which is exactly what committing should be.

The upside is asymmetric, and that asymmetry is the whole argument. What does an allowlist cost you when it's wrong? You forget to un-ignore a real source file, `git add .` skips it, and your CI goes red in about four minutes with "undefined: NewServer." Loud, fast, free to fix — one `!` line. What does the denylist cost you when *it's* wrong? A live key lands in a public commit, gets scraped by a bot within minutes, and — per the same report — **64% of secrets leaked back in 2022 were still valid in 2026.** Nobody rotated them. One failure mode is a caught typo. The other is a credential that outlives the person who leaked it. As one commenter, LaGrange, put it: "recovery from forgetting to add something is much easier than recovery from adding plaintext private keys."

## The one git rule you have to know first

Before you paste that pattern, learn the rule that will bite you, because it's the reason people try the allowlist, watch it "not work," and give up.

From the [gitignore docs](https://git-scm.com/docs/gitignore), exact words:

> It is not possible to re-include a file if a parent directory of that file is excluded. Git doesn't list excluded directories for performance reasons, so any patterns on contained files have no effect, no matter where they are defined.

Read that twice. Once `*` has excluded the directory `internal/`, git will not even descend into it, so `!internal/server/server.go` does *nothing*. The negation never gets a chance to match, because git already decided the whole directory is out and stopped looking.

So you cannot un-ignore your way straight down to a file. You have to re-include the **path**, directory by directory, before you can reach the file inside it:

```gitignore
*
!/internal/
!/internal/server/
!*.go
```

Un-ignore the directory first; *then* the `*.go` inside it can match. This is also why the working allowlist tends to name directories (`!/cmd/`, `!/internal/`) and then filetypes, rather than trying to spell out individual files. When it misbehaves, don't guess — git ships the debugger for exactly this:

```bash
git check-ignore -v internal/server/server.go
```

It prints the precise line in the precise file that decided the outcome, or nothing if the path is tracked. It is the single most useful command in this whole workflow and almost nobody knows it exists. Remember too that within one `.gitignore`, **the last matching pattern wins** — so order matters, and a stray later rule can silently re-exclude something you allowed above.

## The honest case against — and where it holds

The strongest objection is real, and the HN thread made it well. rcfox: "I would 100% forget to unignore files I meant to commit." kryptiskt sharpens it: you won't notice locally, "because it works on their machine" — the file exists on disk, your build is green, and the breakage only shows up in CI or on a teammate's clean checkout. jeltz called the whole idea "terrible advice." Take that seriously. An allowlist moves work from *cleanup* (delete a junk commit) to *vigilance* (remember to enroll every new source file, forever).

Whether that trade is worth it depends entirely on your repo's file-type entropy.

**Where the allowlist wins:** a service with a small, stable set of tracked extensions — a Go or Rust or TypeScript service where 95% of what you keep is `.go`/`.rs`/`.ts` plus a few manifests. New *files* appear constantly; new *kinds* of file are rare. You allow `*.go` once and every future handler is covered. Add "any repository an agent operates in unattended," and it's not close.

**Where it's a tax you'll resent:** a heterogeneous monorepo that legitimately tracks dozens of extensions and picks up new ones every sprint — configs, protos, fixtures, notebooks, generated schemas, images, `.sql`, `.tf`, a new framework's dotfiles. There, the allowlist needs editing so often that the vigilance cost swamps the cleanup cost, and jeltz is right about *your* repo even if he's wrong about the service next door.

There's a self-correcting middle, and cortesoft named it: if a forgotten file breaks the build, CI catches it on the same push. That only holds if your CI actually compiles or runs from a clean checkout — which is the point of CI. A forgotten *source* file is a caught error. The denylist's forgotten *secret* is not caught by anything until it's already public. Design your safety net around which mistake is unrecoverable.

## Don't confuse this with the personal levers

One more thing, because people reach for the wrong tool. Git gives you three ignore surfaces and only one of them protects the *repo*:

- **`core.excludesFile`** (global, `~/.config/git/ignore`) — your personal junk, `.DS_Store`, editor swap files. Never committed, never shared. Good hygiene; protects nobody but you.
- **`.git/info/exclude`** — repo-local, still uncommitted. For files only *you* generate in *this* checkout.
- **The committed `.gitignore`** — the only one that is a shared boundary. It's the one your teammates inherit and the one that governs what the agent-in-CI can stage.

The allowlist flip only matters in that last file. Putting `*` in your global exclude would be a great way to break every repo you own. The boundary you're hardening is the shared one.

## The so-what, for Monday morning

- **Do** flip the committed `.gitignore` to allowlist form on any repo where an agent runs unattended and the tracked filetypes are few and stable — services, libraries, most single-language apps. Lead with `*`, un-ignore directories before filetypes, and keep `!.gitignore` in the list or you'll lock yourself out of your own config. This is a defense against your own `git add .` reflex, which is the reflex the 3.2% number is measuring.
- **Watch** the parent-directory rule. When a file you allowed still won't stage, don't fight the patterns — run `git check-ignore -v <path>` and it'll name the exact line. Nine times out of ten it's an ignored directory swallowing everything beneath it.
- **Ignore** the maximalist framing that this is universal git hygiene. It isn't. On a high-entropy monorepo the allowlist is genuine friction, and the honest answer there is a denylist plus a real secret guardrail — a pre-commit hook or push protection that blocks credentials regardless of which file they're in. The deeper point survives either way: git's default assumes a deliberate human author, and you no longer have one. Whichever list you keep, stop letting "everything, minus what I remembered to exclude" be the rule that decides what your agent publishes.
