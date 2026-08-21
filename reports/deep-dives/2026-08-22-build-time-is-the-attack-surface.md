# The Dangerous Command Was `cargo build`, Not `cargo run`

*Deep dive · Theo Vance (The Builder) · 2026-08-22 · A dependency runs code on your machine before your program ever does — and this week's arrayref attack proved the class is live in the ecosystem people trust most.*

Here is a task you did this week, or your agent did for you. You needed a
fixed-size array reference, so you ran `cargo add arrayref`. Then you ran
`cargo build`. You never ran the program. You didn't even finish writing it.

On Wednesday, that sequence was enough to download and execute an attacker's
binary on your machine, detached, before the compiler emitted a single object
file.

That's the whole story, and it's worth sitting with the shape of it. The
mental model most engineers carry is: *code is dangerous when I run it.* I'll
read the source, I'll check the binary, I'll sandbox the process. The
`arrayref` attack — [published August 20 and pulled from crates.io inside 86
minutes](https://rustsec.org/advisories/RUSTSEC-2026-0260) — runs entirely
inside the step before "run" exists. There is no binary to inspect yet. The
danger was in `build`.

## The mechanism, exactly

`arrayref` is not a fringe crate. It has [~245 million all-time
downloads](https://safedep.io/arrayref-proc-macro1-rust-build-time-malware/)
(244,989,384 at time of writing), and one tracker found it present in
[roughly three-quarters of all environments where Rust is
installed](https://thehackernews.com/2026/08/rust-supply-chain-attack-puts-build.html).
It is the kind of dependency that sits four levels down your tree and you have
never once thought about.

The malicious `0.3.10` release didn't put the payload in `arrayref` itself. It
added a dependency on a crate called `proc-macro1` — a typosquat of
`proc-macro2`, the foundational crate almost every procedural macro in the
ecosystem pulls in. `proc-macro1` carried a `build.rs`.

A build script is the part people forget. [Cargo's own
docs](https://doc.rust-lang.org/cargo/reference/build-scripts.html) are plain
about it: place a file named `build.rs` in a package root and "Cargo will
compile that script and execute it just before building the package." It runs
as a normal program, with your user's full privileges, with no sandbox. The
docs *ask* build scripts to only write to `OUT_DIR` — but that's a style
guideline, not a boundary. Nothing stops a build script from doing anything
your shell could do.

This one, [per Semgrep's and safedep's
teardowns](https://semgrep.dev/blog/2026/rust-crates-arrayref-append-only-vec-compromised-proc-macro1/),
reassembled a base64-obfuscated address at build time, pulled an
architecture-specific binary from `23.254.165.112:9089` over TLS with
certificate verification switched off, wrote it to `/tmp/rust-setup` on Unix
(or a PowerShell dropper to `%TEMP%` on Windows, launched via WScript
specifically to escape Cargo's job object), marked it executable, and spawned
it detached so it outlived the compile. The C2 address was handed to the
payload as an argument. The build dependencies it pulled — a base64 decoder, a
TLS stack, an HTTP client — were the tell; none of that belongs in a crate
that claims to help with tokens.

The account wasn't hacked in the movie sense. The maintainer's credentials
were [likely
compromised](https://blog.rust-lang.org/2026/08/20/supply-chain-attack-on-arrayref/),
and the attacker operated a second account, `dtolney`, one keystroke off
`dtolnay` — David Tolnay, the author of `proc-macro2`, `syn`, and `quote`.
Wiz flagged [significant overlap with known DPRK
campaigns](https://www.wiz.io/blog/rust-supply-chain-attack-on-arrayref-significant-overlap-with-dprk-campaigns).
Three crates were hit the same day — `arrayref`, `internment`,
`append-only-vec` — all removed within two hours, with no evidence anyone
actually got popped.

So the blast radius this time was near zero. That is not the point. The point
is that the class is real, live, and aimed at the ecosystem that markets
itself on safety.

## Why your instincts don't cover this

Rust's whole reputation is *safety*. But memory safety is a property of the
program you run. It says nothing about the code that runs to produce it. The
borrow checker never looked at `proc-macro1`'s `build.rs`, because a build
script is just a program the compiler runs on your behalf, and "on your
behalf" means "as you."

And it isn't only build scripts. Procedural macros are the other build-time
execution surface, and it's the one hiding in plain sight. [A proc-macro crate
is compiled to a dynamic library and loaded into the compiler
process](https://doc.rust-lang.org/reference/procedural-macros.html); it "runs
during compilation, and thus [has] the same resources that the compiler has."
Every `#[derive(...)]` you write is arbitrary code executing inside `rustc`.
The reason the attacker typosquatted `proc-macro2` specifically is that it's
the crate you can hang a malicious dependency off of and have nobody blink.

Notice what this breaks:

- **"I'll read the source before I trust the binary."** There is no binary.
  The code runs to *make* the binary.
- **`cargo build --offline` will save me.** Only if the crate isn't already in
  your local cache. Once it's fetched, offline builds run the same `build.rs`.
- **CI is where untrusted code should run, not my laptop.** Correct — and yet
  your laptop is where you run `cargo add` and `cargo build` to check that the
  thing compiles. The first build is on your machine, with your SSH keys, your
  cloud tokens, your `~/.aws/credentials`.

This is not a Rust bug. It's the universal shape of package managers. `npm`
runs `postinstall` scripts on `npm install` — that's the same door the [npm
worms we covered in
June](../deep-dives/2026-06-10-trust-stack-human-speed.md) walked through, and
it's why npm v12 moved to close install scripts by default. Python runs
`setup.py` at install. Every one of these ecosystems decided, decades ago,
that "installing" and "building" a dependency means "executing its author's
code." Rust just felt like the one place you were safe, and the feeling was
never backed by a mechanism.

## The part that makes this urgent now

Here's the extension that turns a known-old problem into a this-week problem:
the rate at which dependencies enter your tree just went up by an order of
magnitude, because an agent is adding them.

When you ask a coding agent to make something compile, it reaches for a crate,
runs `cargo add`, and runs `cargo build` to check its work — often before you
have read a line of the diff. Anthropic says [~80% of its own merged
code](../deep-dives/2026-07-08-agent-audit-trail-unattended-commits.md) is
model-written (its number, and self-reported). The bottleneck in that world is
review, and review runs at human speed while dependency-adding runs at
copy-paste speed. A `build.rs` payload detonates at the *first build* — which,
in an agent loop, is turn three, long before the pull request, long before a
human looks.

So the trust boundary [moved inside your own
toolchain](../deep-dives/2026-07-17-what-your-coding-agent-sends.md) again. The
coding agent is a networked program holding your keys, and now it's also the
thing choosing your dependencies and triggering the build that runs their
code. `cargo add <something-the-model-suggested>` is not a formatting change.
It is a privileged action, and it should feel like one.

## What actually helps

The ecosystem's fix is real but not shipped-by-default yet. There is an active
[Rust project goal for sandboxed build
scripts](https://rust-lang.github.io/rust-project-goals/2024h2/sandboxed-build-script.html);
a configurable sandbox has landed as an unstable Cargo feature, opt-in first,
with the plan to flip it on by default in a future Edition. `proc-macro2`'s own
author wrote [Watt](https://github.com/dtolnay/watt) *years* ago — a runtime
that executes procedural macros as sandboxed WebAssembly, where a macro
"[can't] do anything other than shuffle tokens around." The defense has
existed longer than the attack. Almost nobody turned it on.

Until the defaults change, this is a discipline, not a setting.

**Do (this week):**

- Treat a new dependency as untrusted code that runs at build time, because it
  is. Before the *first* build of anything new — especially something an agent
  added — skim its `build.rs` and its proc-macro deps for network or process
  calls. A build dependency on an HTTP client is a smell.
- Run first builds of new/updated crates somewhere that isn't holding your
  secrets: a container, a throwaway CI job, a sandbox with no network egress
  and no credentials mounted. If a build needs to phone home, make it fail.
- Commit and pin your `Cargo.lock`, and gate updates. Set `cargo build`,
  `cargo add`, and `cargo update` as actions your agent must ask for, the same
  way you'd gate a `curl | sh` — [a PreToolUse hook or a permission rule that
  requires confirmation](../deep-dives/2026-07-23-sandbox-is-the-real-brake.md),
  not blanket allow.
- Wire `cargo audit` / `cargo deny` into CI against the [RustSec advisory
  DB](https://rustsec.org/advisories/RUSTSEC-2026-0260) so a known-bad version
  fails the build instead of running it. For a security-critical tree, adopt
  [`cargo-vet`](https://mozilla.github.io/cargo-vet/) and require a human audit
  before a crate enters — high friction, and that's the point.

**Watch:**

- The sandboxed-build-script feature stabilizing, and the Edition that turns it
  on by default. That's the day this stops being your problem and starts being
  the toolchain's.
- Typosquats of *foundational* crates (`proc-macro2`, `serde`, `syn`), which is
  where this attacker aimed. A one-character dependency name in a lockfile diff
  is now a security event.
- Whether your agent harness logs and gates dependency changes as
  first-class actions, or buries them in a green "build succeeded."

**Ignore:**

- "Rust is memory-safe, so this can't happen." Memory safety and supply-chain
  safety are unrelated properties. This attack never touched unsafe code.
- The specific IP and file paths in this week's write-ups. They're one
  campaign's infrastructure; they'll rotate. The reusable fact is the surface:
  code runs at build time, with your privileges, with no sandbox, today.

The so-what for Monday morning is small and concrete. Find the one place in
your workflow — your laptop, most likely — where the first build of a brand-new
dependency happens with your real credentials in the environment. Move it. That
single change turns the most likely version of this attack from "detached
payload with my AWS keys" into "a build that failed in a box." Everything else
is hygiene you already know how to do; you just have to start treating `build`
as the command that runs untrusted code, because it is.
