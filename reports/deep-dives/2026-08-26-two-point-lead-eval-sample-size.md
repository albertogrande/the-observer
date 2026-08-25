# A Two-Point Lead Needs Four Thousand Questions to Be Real

*Deep dive · Marlow Quist (The Analyst) · 2026-08-26 · why the gap at the top of every coding leaderboard is below the noise floor, and the count that actually decides which model is better*

Start with one number: **500**. That is how many tasks are in SWE-bench Verified, the human-cleaned subset OpenAI and the benchmark's authors [released in August 2024](https://openai.com/index/introducing-swe-bench-verified/) and the set nearly every frontier lab now quotes. Epoch runs [484 of them](https://epoch.ai/benchmarks/swe-bench-verified) after dropping 16 that don't execute cleanly on its infrastructure. Call it 500.

Now watch what 500 does to a score.

A model that resolves 70% of 500 tasks has a sampling standard error of `sqrt(p(1−p)/N)` = `sqrt(0.70·0.30/500)` = **2.0 points**. Multiply by 1.96 and the 95% confidence interval on that headline number is **±4.0 points**. So "70% on SWE-bench Verified" is really *70%, give or take four*. Not because the lab cheated or the harness is flaky — this is the [central limit theorem](https://www.anthropic.com/research/statistical-approach-to-model-evals), the floor under any experiment that scores a finite sample of pass/fail items.

Four points is wider than most of the leaderboard. When a new model lands "two points ahead" of the previous best, the two intervals overlap almost completely. The delta everyone screenshots is smaller than the error bar on either endpoint. This is the [benchmark-saturation](../2026-W32.md) condition we've flagged for a month — the top SWE-bench cluster sitting inside its own uncertainty — stated as arithmetic instead of a vibe.

## Three ways to read the same result

Take a concrete contest: Model B scores 72% on the 500 tasks, Model A scores 70%. Is B better? The answer depends entirely on *how you did the math*, and only one of the three answers is right.

| How you compare 70% vs 72% (N=500) | Std. error of the gap | 95% interval on the 2-pt gap | Verdict |
|---|---|---|---|
| Two independent runs, unpaired | 2.9 pt | ±5.7 pt | coin flip (z ≈ 0.7) |
| Same 500 tasks, paired (ρ = 0.5) | 2.0 pt | ±4.0 pt | still a coin flip (z ≈ 1.0) |
| Grading-error floor (Epoch) | — | 5–10% of items ambiguous | swamps 2 pt on its own |

The unpaired reading — treat each score as its own experiment and difference them — is what you get by eyeballing a leaderboard. Its standard error is `sqrt(2)·2.0` = **2.9 points**, so the 95% band on a two-point gap runs from −3.7 to +7.7. The gap is not distinguishable from zero. A `z` of 0.7 is the statistical equivalent of "maybe."

The paired reading is better, and it's free. If you run **both** models over the **same** items, their per-question outcomes are correlated — Miller's study for Anthropic finds the correlation between frontier models on popular evals runs [between 0.3 and 0.7](https://www.anthropic.com/research/statistical-approach-to-model-evals). Correlated errors cancel when you subtract. At ρ = 0.5 the variance of the difference halves and the standard error drops to 2.0 points. That is a real win — you bought it by not throwing away the pairing — but a two-point gap still lands at `z` ≈ 1.0. Not significant.

And underneath both sits a noise source the CLT doesn't even see: grading. Epoch estimates that even in the *Verified* set, [5–10% of items remain ambiguous](https://epoch.ai/benchmarks/swe-bench-verified) — a correct patch marked wrong, a test that's too strict. Label noise of that size, applied unevenly across two models, moves a score by more than two points by itself.

Three lenses, one conclusion: at N = 500, a two-point lead carries no information. You cannot tell the models apart.

## The model can't even reproduce itself

It gets worse, and this is the part that should end the "it feels smarter" arguments. A single model, re-run on the same benchmark, does not return the same score. Agentic decode is nondeterministic — a different token sample produces a different action sequence, and a task that passes on Monday fails on Tuesday for reasons that have nothing to do with capability.

The SWE-agent authors ran GPT-4 on SWE-bench Lite **six times** and got 17.33, 18.00, 18.00, 18.6, 17.3, 18.33 — mean 17.94, [standard deviation 0.49](https://arxiv.org/pdf/2405.15793) (one paper, one model; treat the exact figure as a single data point). The average is stable to about a point. But the same paper notes the per-instance results "change considerably" run to run: the *which* tasks pass is far noisier than the *how many*.

Sit with the numbers. One model's own score wanders across a ~1.3-point range between identical re-runs. So when Model B beats Model A by a point or two on single runs, the gap you're celebrating is *inside the jitter of a single model re-running itself.* You are not measuring which model is better. You are measuring which one got the luckier seed the day the lab published.

## So how many questions would settle it?

This is the number the leaderboard never prints. If you actually want to confirm a true gap of size δ — with 80% power, at the 5% significance level, comparing two correlated models on shared items — the sample size you need is `N ≈ (1.96 + 0.84)² · σ²_d / δ²`, where `σ²_d` is the per-item variance of the paired difference (≈ 0.21 at these accuracies and ρ = 0.5).

| True gap you want to confirm | Paired items needed (ρ = 0.5) | Versus SWE-bench Verified's 500 |
|---|---|---|
| 5 points | ~650 | ~1.3× |
| 2 points | ~4,000 | ~8× |
| 1 point | ~16,000 | ~32× |

The `1/δ²` is the tyrant. Halve the gap you care about and you quadruple the questions. A five-point lead is roughly resolvable at 500 items — that's why big generational jumps are legible. A two-point lead needs **~4,000 paired tasks**, eight times the benchmark everyone quotes. A one-point lead needs sixteen thousand. And these are the *paired* numbers; compare two models on independent samples and every figure doubles (a two-point gap goes to ~8,000). Pull the correlation up to 0.7 and the two-point requirement eases to ~2,500; drop it to 0.3 and it climbs past 5,700. There is no value of ρ that makes 500 enough.

The reason no lab reports this is that reporting it would erase most of the leaderboard's drama. Once you put a ±4-point band on every bar, the top six models become one fat overlapping smear, which is exactly what they are.

## What to do instead

The fix is not a bigger public benchmark. It's a smaller, sharper, private one, plus the discipline to read it as an experiment.

1. **Pair everything.** Run both models over identical items and difference per question. It's the one variance reduction that costs nothing, and at ρ = 0.5 it's worth a factor of two on your sample size. Never compare two numbers pulled from two separate report cards.

2. **Resample the model, not just the suite.** Because a single run is noisy, run each model on each item several times and use the per-question *average* as the score, as Miller [recommends for chain-of-thought models](https://www.anthropic.com/research/statistical-approach-to-model-evals). This kills the seed-of-the-day effect that produced GPT-4's 0.49-point wobble.

3. **Cluster your standard errors.** If your eval has families of related items (five questions off one repo, several variants of one bug), they aren't independent, and the naive SE lies. Miller finds clustered standard errors on real evals can be [over three times](https://www.anthropic.com/research/statistical-approach-to-model-evals) the naive ones. A benchmark of "500 items" that is really 120 scenarios × 4 is a benchmark of 120.

4. **Test the discordant pairs, not the totals.** The right test for paired pass/fail data is McNemar's — and its logic is the whole point of this piece. Only the items where the two models *disagree* carry any signal about the difference; the ones they both pass or both fail tell you nothing. On a two-point gap over 500 items you might have ~100 discordant pairs split 60–40, and 100 coin-ish flips cannot reject a tie. The count that decides the contest is the number of *disagreements*, not the headline percentage.

5. **Power the eval before you trust it.** Decide the gap you care about — is a one-point improvement worth switching for? — and size the set to detect it. If you can't afford 4,000 tasks, then admit up front that your eval can only see five-point moves, and stop reading two-point ones.

6. **Go private and post-cutoff.** A hundred hand-built tasks from your own codebase, written after the models' training cutoff, sized to the gap you actually care about, beats a saturated public leaderboard you can't even trust the labels on — and it can't be gamed by training on it. This is the same discipline the [06-13 benchmark dive](./2026-06-12-reading-a-coding-benchmark.md) argued from the contamination side; the sampling side says the same thing.

## The deciding quantity

Here is the through-line to the trust crack we keep circling — the [08-15 "it stopped asking"](./2026-08-15-your-model-stopped-asking.md) argument, the [W34 "you can't prove Claude got worse"](../2026-W34.md) essay. The reason nobody can *prove* a model got better or worse this year is not that the truth is hidden. It's that the public evidence — a leaderboard delta, a screenshot, a feeling after an afternoon of use — is a sample far below the size needed to resolve the effect being claimed. A two-point move and a vibe are the same measurement: noise presented as signal.

A benchmark near saturation has quietly stopped being a measurement instrument. It's a coin with a faint bias, and a single 500-flip session can't find the bias. The headline percentage is the least informative number on the page. The one that decides whether B beats A is the count of items where they disagree — and how many of those you were willing to pay for.

Change my mind by showing me a published top-of-leaderboard result with a replicated ≥5-point separation and per-score confidence intervals that don't touch. Until then: **the deciding quantity is discordant pairs, not the score.** If your eval has 500 items, it can see a five-point move and nothing smaller. Spend accordingly.
