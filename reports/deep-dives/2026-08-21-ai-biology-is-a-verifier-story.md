# Claude Designed a Working Drug This Week. It Also Couldn't Have Run the Trial.

*Deep dive · June Okafor (The Contrarian) · 2026-08-21 · Everyone read the protein-binder result as "AI does biology now." Two biology headlines landed the same week, at opposite ends of the one variable that actually decides where AI wins.*

Here is the sentence everyone repeated this week. **AI is doing real biology now.**
Not predicting structures, not writing up other people's experiments — designing
working molecules, confirmed in a wet lab, beating human experts. Anthropic ran an
autonomous protein-design campaign and Claude
[hit 14 of 15 targets](https://www.anthropic.com/research/Claude-accelerates-protein-design).
Drug discovery, the take went, is next.

I believe the result. I want to break the reading of it.

Take the steelman at full strength, because it is strong. This was not a demo.
Claude's models produced **1,320 designs and 354 confirmed binders across 14 of 15
targets**, with **minimal human involvement beyond the initial prompt** — Anthropic
says it gave "no additional scientific, technical, or operational guidance" once the
campaign started. The hit rates are the headline: **26.7%** for Mythos Preview and
**22.6%** for Opus 4.8 designing against all targets at once, rising to **35.1%** when
Mythos Preview focused on one target at a time — against an industry baseline Anthropic
puts at **10–15%**. Two independent labs,
[Adaptyv Bio](https://www.adaptyvbio.com/blog/po104/) and Twist Bioscience, actually
made the proteins and measured them. Some of Claude's best designs bound *tighter* than
the best previously published result. And on a separate task, Opus 5 read raw NMR and
LC-MS instrument files from a two-sentence prompt and returned a structure and a purity
figure — **96.4%** against the lab's **96.33%** — in about twenty minutes. That is not
hype. That is a machine doing bench science.

So why am I not writing the "AI does biology" piece?

## What actually did the work

Because the thing that made this work is not that Claude understands biology. It is that
**protein binding is the cheapest verifier problem in the life sciences.**

A binder is a molecule that sticks to a target. "Sticks" is not a matter of opinion. You
express the candidate protein, run it on an automated
[bio-layer interferometry](https://docs.adaptyvbio.com/experiment-types/binding-screening)
pipeline, and read off a number — the dissociation constant, K_D. Below 10 nanomolar,
Anthropic counted it a high-affinity hit. Adaptyv's platform quantifies K_D from 0.1 nM to
10 µM, thousands of designs in parallel, on a screening turnaround of about **three weeks**.
Cheap. Fast. Faithful. Massively parallel.

That changes what "Claude designed a binder" means. The model is a proposal engine. The
assay is the judge. Generate 1,320 candidates; the bench throws away the ~73% that don't
bind and keeps the 354 that do. Nobody had to be right up front. They had to be checkable
after the fact, at volume, quickly.

We have a name for this on the software side of the house. It is the
[verifier asymmetry](./2026-07-24-verifier-asymmetry-check-vs-find.md): AI wins the tasks
where a cheap, runnable check exists, because you can let it search and let the checker
sort the output. A binding measurement is a *certificate* — one physical experiment settles
a candidate, the way a single counterexample settles a math conjecture or a passing test
settles a patch. The same law
[predicted where AI would level up a novice and where it wouldn't](./2026-08-09-ai-levels-where-errors-are-cheap.md):
cheap verifier, machine supplies the check, everyone rides along; expensive verifier, only
the expert can supply the check, and the leverage evaporates. Protein binding sits at the
cheap-and-faithful extreme. That is why it fell first — not because the model crossed some
threshold of biological insight.

And the in-silico half of the campaign is the same story one level down. Claude did not
imagine molecules from nothing; it orchestrated "publicly available specialist protein
design and co-folding models" and ran candidates through "multiple cycles of in silico
optimization." Structure prediction is itself a cheap *approximate* verifier — it pre-scores
designs before anyone spends a pipette on them. The wet lab is the faithful verifier that
ratifies what the cheap one guessed. It is verifiers all the way down. The intelligence in
the loop is real, but it is intelligence *pointed at a gradable target*.

## The other biology headline

Here is where the week did me a favor. The protein result was not the only biology story on
the front page. Two days earlier, Moderna and Merck said their personalized mRNA cancer
vaccine had cleared a
[Phase 3 trial](https://www.merck.com/news/merck-and-moderna-announce-phase-3-interpath-001-trial-of-intismeran-autogene-plus-keytruda-met-endpoints-of-recurrence-free-survival-rfs-and-distant-metastasis-free-survival-dmfs-in-patient/) —
the first ever for an individualized neoantigen therapy. It is the same field. It is at the
opposite end of the only variable that matters.

Intismeran autogene is, mechanically, an AI-adjacent design problem: you sequence a patient's
tumor, find its mutational fingerprint, and print synthetic mRNA coding for up to **34
neoantigens** tailored to that one person. Picking those 34 is a modeling task. But that is
not what took eight years and hundreds of millions of dollars. What took the time was the
*verifier*. The question for a cancer vaccine is not "does this molecule bind?" It is "does
this keep a real human alive and cancer-free?" — and the only instrument that answers it is a
randomized Phase 3: **1,137 patients**, randomized **2:1**, measured on recurrence-free and
distant-metastasis-free survival, which you observe by *waiting for real people over real
years*. INTerpath-001 was launched in 2023, on a program whose earlier-phase readouts stretch
back to a trial that has now reported five-year data. No model compressed that. You cannot
parallelize a calendar. You cannot run best-of-64 on a human being.

So the same week showed both ends of the same field. Where the oracle is a three-week bench
assay, Claude produced a 2–3× leap over the human baseline, autonomously. Where the oracle is
a human body over years, biology moved at exactly the speed it has always moved, and the win
belonged to the people who spent the decade running the trial.

## The counter-thesis

That is the pattern, and it is my claim: **AI's biology wins will track the cost and fidelity
of the verifier, not the intelligence of the model.** Point a generate-and-check loop at a
problem with a cheap, faithful, fast oracle and you should expect protein-binder-shaped
results — more, and sooner. Point it at a problem whose only honest oracle is expensive, slow,
and un-parallelizable, and the model helps at the edges — pick the neoantigens, read the
assay, draft the protocol — while the timeline stays governed by the check.

And the hype cycle will do exactly one thing wrong: it will slide from the first sentence to
the second. From "designed a binder that binds in a dish" to "curing disease." Those two claims
are separated by the most expensive verifier in all of science, and the slide is where credulity
gets manufactured.

Let me take the strongest objections head-on, because a couple are good.

**"Reading raw NMR and LC-MS and iterating — that's understanding, not screening."** It is
genuinely impressive. But look at *why* you can trust the number: hydrogen count within 0.08
of the truth, purity 96.4 against a measured 96.33. Those are checkable against the physical
sample in one pass. The analytical-chemistry win is not a counterexample to the verifier story;
it is another instance of it. It is a task where the answer is cheap to grade, which is exactly
where the model shines.

**"Structure prediction did the heavy lifting — the wet lab barely mattered."** Partly true,
and it makes the point stronger, not weaker. The campaign still manufactured 1,320 real molecules
and discarded nearly three-quarters of them. Cheap approximate verifier up front, faithful
verifier at the end; the whole apparatus is a filter, and filters need something to filter
against.

**"AI will compress the clinical verifier too — digital twins, synthetic control arms, surrogate
biomarkers."** This is the one to watch, and it is also the trap. Those techniques improve trial
*efficiency* — enroll fewer, read out sooner — but they do not remove the irreducible need to
observe real outcomes in real people, and every surrogate endpoint is a *cheaper proxy verifier
that can be wrong*. Recurrence-free survival is already a surrogate for overall survival, and
regulators argue about how faithful it is. Swapping a faithful-but-expensive check for a
cheap-but-approximate one is not a free lunch; it is the Goodhart move, and the verifier-asymmetry
law says a gamed proxy is where the whole thing quietly breaks.

## So what, if you don't design proteins

The rule is domain-agnostic, which is the only reason it belongs in a dev publication. Before you
point an agent — or a whole autonomous campaign — at anything, ask four questions about the check,
not the model. What is the verifier? What does it cost to run? How fast does it return a verdict?
And is it faithful, or a gameable proxy? Binding assays, unit tests, compilers, and type checkers
are the cheap-and-faithful end, and that is where you should expect an agent to earn its keep by
volume. "Is this the right architecture," "is this secure," "will this drug work in humans" are the
expensive-or-unfaithful end, and there the ceiling on AI is the price of the check, not the size of
the model. Your yield is set by which end your verifier sits on. When you next read "AI solved X in
biology," the real question is not how smart the model is. It is: *what checked it, and how cheap was
the check?*

Here is what would prove me wrong. If a de-novo, AI-designed molecule reaches a Phase 3 clinical
endpoint on a materially compressed timeline — and the compression comes from the *model* rather than
from trial logistics — then the expensive verifier itself got cheap without losing fidelity, and my
line collapses. Or if AI binder hit-rates hold at 2–3× the human baseline on targets that have *no*
cheap binding assay — intrinsically disordered proteins, membrane receptors in their native context —
then the model is generalizing past the reach of the verifier, and this was never a verifier story to
begin with. Until one of those lands, the shape holds: AI is a proposal engine chained to a checker,
and biology's ceiling on it is the price of its proof.
