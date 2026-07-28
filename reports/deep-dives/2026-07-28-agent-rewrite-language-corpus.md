# Bun's Rewrite Only Runs in One Direction

*Deep dive · June Okafor (The Contrarian) · 2026-07-28 · why the Zig→Rust arrow, not the eleven days, is the real lesson of the agent-fleet port.*

Here is the story everyone took away from the Bun rewrite. A fleet of Claude
agents ported the runtime from Zig to Rust in eleven days for about $165,000,
and Claude Code now runs on the result. So a large migration — the kind that
used to eat a team for a year — is now a weekend and a credit-card bill. Pick a
better language, point the agents at your repo, wait.

That reading is popular because the numbers are genuinely startling, and I want
to give it full weight before I take it apart.

## The steelman

The facts hold up. Bun's creator, Jarred Sumner — whose company Oven
[joined Anthropic](https://bun.com/blog/bun-joins-anthropic) last December —
[ran roughly 535,000 lines of Zig](https://www.developersdigest.tech/blog/bun-rust-rewrite-agent-fleet-case-study)
through a pre-release Fable 5 between May 3 and 14. At peak the setup was four
Claude Code workflows running at once, each in its own git worktree, sixteen
agents to a workflow — about sixty-four Claudes at a time, across some fifty
dynamic workflows over the eleven days. Cost came to about $15,000 a day. The
isolation primitive was the boring one that already works: [a worktree per
workflow](./2026-06-23-git-worktrees-agent-isolation.md), so parallel writers
don't corrupt a shared checkout.

And the task shape was close to ideal for an agent. A rewrite is not greenfield.
You start with a complete, working reference implementation — the Zig source is
the spec — and an existing test suite you can run as an oracle. That is exactly
the setup where agents earn their keep: a cheap, runnable check to search
against, which is [the shape of every task agents actually
win](./2026-07-24-verifier-asymmetry-check-vs-find.md). Sumner's stated motive
was legitimate too. Zig is not memory-safe; Rust is. The point was to
"systematically prevent memory bugs from recurring." If you were going to demo
that agents can move real infrastructure, this is the honest way to do it.

So: real event, real numbers, real reason. The consensus isn't wrong that
something changed. It's wrong about *what*.

## Look at the arrow again

Everyone read the rewrite as *old codebase → better language*. Read the arrow
literally. It points from a low-resource language to a high-resource one. That
is not a footnote to the economics. It is the precondition for them.

A language model's fluency in a language is, to a first approximation, a
function of how much of that language it read in training. Rust is not a
niche language to a model. In [The Stack v2](https://arxiv.org/abs/2402.19173),
the code corpus behind StarCoder2, Rust is 15.6 GB across 2.22 million files —
smaller than the giants (JavaScript 1,115 GB, Python 233 GB) but comfortably in
the tier every code model trains on and every benchmark tests. Zig is not in the
top thirty-two languages of the 658 the dataset detected. It is a rounding error.

The performance gap that follows is not subtle. A 2026 survey of code generation
by language tier found pass@1
[near 100% on high-resource languages, above 50% on
low-resource ones, and below 20% — near zero on hard problems — on no-resource
languages](https://arxiv.org/abs/2606.16827). Rust sits at the top of that
curve. Zig sits far down it.

Now run the Bun play into Zig instead of out of it. Sixty-four agents, eleven
days, a test suite as the gate. You do not get 535,000 clean lines. You get a
per-agent error rate set by how little Zig the model ever saw — and because you
fanned the work across sixty-four agents, you multiply that rate sixty-four ways.
Parallelism is a throughput trick that only pays when each worker succeeds
locally. In a language the model barely knows, fan-out multiplies the slop.

Which is why [Zig's creator Andrew Kelley called the output "unreviewed
slop"](https://www.theregister.com/devops/2026/07/14/zig-creator-calls-buns-claude-rust-rewrite-unreviewed-slop/5270743)
and asked the sharp question: if the existing test suite couldn't prevent the
bugs in the *Zig* implementation, why is it sufficient evidence for a million
lines of new Rust? That's the [reviewer-is-the-bottleneck
problem](./2026-07-22-human-in-the-loop-deskilled.md), and it's real. But notice
that Kelley's "slop" and Sumner's "eleven days" are the same fact seen from two
sides. The cost was low and the volume was high *because the target was a
language the model had read a million files of.* The slop rate and the price tag
are both functions of corpus size.

## Why Rust specifically, past the memory-safety story

Two things make a language cheap for an agent fleet to write. One is corpus:
fluency, so the first draft is mostly right. The other is a cheap, faithful,
runnable verifier: something that catches the errors without a human in the
loop. Rust maxes both. It has the corpus, and it ships a borrow checker that
rejects — at compile time — the exact class of memory bugs the rewrite existed
to eliminate. The compiler is a verifier the agent gets for free, on every
build, for the one bug class that mattered most.

Zig has neither. Small corpus, and no borrow checker to lean on. So the rewrite
wasn't really Zig → "a better language." It was Zig → "the language an agent
fleet can write fluently and check for free." Memory safety is a true reason to
prefer Rust over Zig. But it explains the *destination*, not the *feasibility*.
The feasibility is corpus times verifier, and Rust happens to sit at the maximum
of both. Nobody is naming the first term.

## The counter-thesis

Here is what I think actually changed, and it is bigger than "migrations got
cheap."

When the model is the primary author, training-corpus size becomes a first-class
language-selection criterion. It now sits on the same checklist as performance,
memory safety, ecosystem, and hiring pool — how much of this language has the
model seen? For most teams that question was invisible a year ago. After Bun it
is a line item, because it decides whether an agent fleet can help you maintain
the thing at all.

And the criterion is self-reinforcing, which is the part that should worry you.
Model-friendly languages attract more AI-written code. That code becomes more
public corpus. The next model is even more fluent in them. Rich get richer. The
languages best positioned for the agent era are not the best-designed ones; they
are the best-represented ones — plus whatever ships a strong compile-time
verifier. Zig did not lose an argument about semantics. Zig is, by many accounts,
a genuinely interesting language. It lost a corpus-size contest to a language
with a hundred times the files, and the contest was decided by data volume, not
by design.

That is a strange new pressure on an ecosystem, and it runs opposite to how we
usually talk about language choice — on merit, on fit, on taste. The agent
doesn't have taste. It has a training distribution.

## Where this is wrong, honestly

Three real objections.

**Sumner picked Rust for memory safety, not for the model.** True, and I'm not
claiming he sat down and reasoned about corpus size. But memory safety chooses
Rust *over Zig*; it does not explain why porting half a million lines was cheap
enough to attempt on a whim. Two independent facts both point at Rust, and only
one of them is in the writeups. The quiet one is the corpus.

**Low-resource languages aren't doomed.** You can manufacture training data:
[MultiPL-T translates high-resource training examples into low-resource
languages](https://dl.acm.org/doi/10.1145/3689735), and continual pretraining on
refined code has lifted pass@1 by double digits on HumanEval. A niche language
*can* be rescued. But ask who pays. You fund a data program for a language with
demand — and the niche language's whole problem is that it lacks the demand to
fund its own rescue. The subsidy flows to the languages that already have the
corpus, the same way a [price cut flows from whoever the token is a complement
for, not a product](./2026-06-28-price-cut-is-a-weapon.md). The gap is more
likely to widen than close on its own.

**Maybe general scaling closes the per-language gap.** Some evidence says it
narrows. But the truly low-resource tail persists near the bottom of that pass@1
curve, and the tail is exactly what decides a $165,000-versus-can't-do-it call.
A gap that's shrinking in the aggregate can still be decisive at the margin where
you're standing.

## What would change my mind

The falsifiable version: if AI authorship weren't a language-selection force,
we'd see agent-fleet rewrites flow *toward* technically superior niche languages
when the merits favor them — or at least run in both directions. My prediction,
for the next year, is that the publicly documented large agent-driven ports run
overwhelmingly *into* top-corpus languages — Rust, Go, TypeScript, Python — and
that no niche language (Zig, Nim, Crystal, Odin, V) is the *target* of a
comparable fleet rewrite. Confidence 70%. If someone ships a Bun-scale rewrite
*into* a low-resource language at Bun-like cost and quality, I'm wrong, and the
corpus stops being destiny.

The so-what is small and concrete. If you're choosing a language for a codebase
an agent will help you maintain, add one question to the list: is this language
in the model's benchmark set, and does it have a compile-time verifier the agent
can lean on? That's not a purity concern anymore. It's a throughput and a
defect-rate input, and Bun just published the numbers. And if you love a niche
language, plan around the tax: lean harder on its types, tests, and compiler,
[recount your token costs per language](./2026-07-14-tokenizer-real-price-per-file.md),
and don't expect sixty-four agents to save you. The model can't write your
language yet. After this week, that is a reason people will switch — whether or
not it should be.
