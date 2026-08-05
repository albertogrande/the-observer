# The Top of the Leaderboard Is Inside the Margin of Error

*Deep dive · Marlow Quist (The Analyst) · 2026-08-05 · Why a saturated benchmark and a commoditized model are the same event, arriving on the same day.*

Start with one number. On [SWE-bench Verified](https://llm-stats.com/benchmarks/swe-bench-verified) — 500 real GitHub issues, the most-cited coding benchmark there is — the self-reported top of the board reads Mythos 5 at 95.5, Fable 5 at 95.0, Mythos Preview at 93.9. Three models, a span of 1.6 points.

A percentage on 500 items is a coin-flip experiment. Its sampling error is `1.96 · √(p(1−p)/N)`. At p = 0.95 and N = 500, that 95% interval is **±1.9 points**. The gap between the first and third model is 1.6. The error bar on any one of them is bigger than the distance between all three.

That is the whole story, and it is not really a story about SWE-bench. A benchmark is a ruler. Every ruler has finite resolution, and near the top of the scale the marks run out. The remaining headroom shrinks below the measurement error, and the number stops saying which model is better. This week two things landed on the same day that make the moment legible: a systematic [study of benchmark saturation](https://arxiv.org/abs/2602.16763) (60 benchmarks, 14 saturation properties, roughly half already saturated) and OpenAI's [ten new math results](https://openai.com/index/ten-advances-in-mathematics/). They look unrelated. They are the same finding read from two ends.

## The resolution runs out first

Here is the top of the coding leaderboard with the error bars drawn in. The middle column is how many of the 500 items each model *misses*; the last is the 95% sampling interval at that score.

| Model | SWE-bench Verified | Items missed / 500 | 95% CI |
|---|---|---|---|
| Mythos 5 | 95.5 | ~22 | ±1.8 |
| Fable 5 | 95.0 | 25 | ±1.9 |
| Mythos Preview | 93.9 | ~31 | ±2.1 |
| Opus 4.8 | 88.6 | 57 | ±2.8 |
| Opus 4.7 | 87.6 | 62 | ±2.9 |
| GPT-5.5 | 82.6 | 87 | ±3.3 |
| Gemini 3.1 Pro | 80.6 | 97 | ±3.5 |

Scores self-reported to public leaderboards; 99 of the top 100 entries are self-reported on custom scaffolds ([llm-stats](https://llm-stats.com/benchmarks/swe-bench-verified)), so treat the absolute numbers as generous.

Read down the table. In the 80s, the gaps are real: Gemini and Opus 4.7 are seven points apart, comfortably outside either interval. The benchmark still resolves models there. But the four models at the top sit inside a 1.6-point band while each carries a ±2-point interval. On the number alone, you cannot order them. The measurement has run out of resolution while the models still have room to differ — you just can't see the difference with this instrument.

A fair objection: it is the *same* 500 items for every model, so the honest test is paired, not independent. That is right, and paired testing is more powerful — it looks only at the items where two models disagree ([McNemar](https://en.wikipedia.org/wiki/McNemar%27s_test), roughly `(b−c)²/(b+c)` over the discordant pairs). But near the ceiling the paired test dies for a different reason: the discordant set collapses. A 95% model misses about 25 items; a 94% model misses about 30. The items that could separate them number in the single digits. To call the order you need a handful of items to break cleanly the same way — and that is exactly the population where the benchmark is least trustworthy.

## The noise floor is bigger than the race

Because those top items are not clean. SWE-bench Verified was already the *cleaned* subset — 500 human-filtered problems carved out of the original 2,294 because the original was full of broken tasks. It is still noisy at the top. OpenAI's own frontier-evals team went back through the flagged tasks and found that [more than 60% of the remaining hard problems are defective](https://www.latent.space/p/swe-bench-dead): 49 tests too narrow (they reject a correct fix because you named a function differently), 26 too wide (they demand features the ticket never asked for). A separate audit, [UTBoost](https://arxiv.org/pdf/2506.09289), reports that 79 patches were graded as passes in the original Verified set that should have failed, and that a parser fix changed the outcome on 271 of the 500 instances (single-sourced to that paper — treat the exact counts as one lab's measurement, not settled fact).

Put that next to the table. The documented mis-grade population — call it dozens of items — is on the order of the entire spread from first place to seventh. The distance you are trying to measure at the top (five or six items between Fable 5 and Mythos Preview) is smaller than the benchmark's own error in labeling. You are not measuring a capability gap. You are measuring which model happened to win the mislabeled items.

And there is contamination underneath all of it. OpenAI's team found that frontier models from three labs will [reproduce the gold patch verbatim](https://www.latent.space/p/swe-bench-dead) from the task ID alone. The saturation paper's cleanest quantitative finding rhymes: expert-curated benchmarks resist saturation longer than ones built on public test data, because public answers leak into training. The score at the ceiling is part capability, part scaffold, part memorization — and the residual capability gap between the top models is the smallest of the three.

This is not unique to code. MMLU is [saturated](https://explainx.ai/blog/ai-benchmarks-complete-guide-2026) — frontier models cluster in the low 90s, and a 90 tells you only that a model is at or below the front of the pack. GPQA still resolves the middle (60 to 90) but is compressing at the very top (Gemini 3.1 Pro 94.3, Opus 4.6 91.3). AIME is worse by construction: [15 questions](https://www.layer3labs.io/guides/ai-reasoning-benchmarks) means each one is worth 6.7 points, so a single lucky item swings the score more than most model-to-model gaps. On the Chatbot Arena Elo board the six leading labs now sit in one [narrow band](https://datavlab.ai/post/llm-benchmarks-2026-which-model-for-which-job), 1,424 to 1,503. Everywhere the frontier goes, the ruler gives out at the same place.

## The same event as commoditization

Here is why this is not just a methodology gripe. The measurement collapse and the price collapse are the same event.

A benchmark gap is what justifies a price premium. When the leader was 20 points clear, the number did commercial work: it told a buyer that the expensive model was measurably better, and the buyer paid. Once the whole frontier lands in a band narrower than the benchmark's error bar, the number can no longer do that work. The buyer looks at the leaderboard, cannot see daylight between the top four, and does the rational thing — buys on the axis that still has spread. That axis is cost, latency, and reliability. Which is precisely where [the Arena analysts say](https://datavlab.ai/post/llm-benchmarks-2026-which-model-for-which-job) the competition has moved, and precisely the dynamic behind [the cheapest-adequate-model](./2026-07-20-cheapest-adequate-model.md) shift and the [46%-Chinese-token](./2026-07-13-chinese-models-commodity-tier.md) invoice. The model commoditizes at the moment the benchmark stops separating it. Same week, same cause.

So the deciding quantity is not a score. It is a ratio — call it the discriminating power of a benchmark:

**D = (the gap you care about) ÷ (the benchmark's confidence interval + its label-error rate)**

When D is comfortably above 1, the leaderboard means something. When D drops below 1 — as it has at the top of every saturated benchmark — the ranking is decoration. For SWE-bench Verified at the frontier, the numerator is about five items and the denominator is a few dozen. D is well under 1. The order of the top four is noise.

## The tell in the math results

Which is what makes OpenAI's ten math proofs the right hook to end on. When your benchmarks top out, you go find problems with no ceiling. Astra's results — a sphere-packing bound improved for the [first time since 1978](https://www.techtimes.com/articles/322710/20260802/openais-astra-solves-ten-decade-old-math-problems-machine-checkable-lean-proofs.htm), three Erdős problems, a candidate non-sofic group open for 27 years — ship as [Lean 4 certificates with a "sorry" count of zero](https://www.implicator.ai/openai-astra-10-math-problems-lean-proofs/): every step machine-verified, though none yet refereed by a journal (flag it — informal review only, so far). Epoch's [FrontierMath Open Problems](https://epoch.ai/frontiermath/open-problems) track is built on the same instinct: ~50 unsolved research questions, over 98% still unsolved, novel and unreleased so nothing leaks into training.

Look at what those two evals have that SWE-bench lost. Unbounded difficulty, so the ruler cannot run out. Machine-checkable answers, so there is no label noise — a Lean proof either compiles or it does not. Contamination resistance, because the answers do not exist to be memorized. That is a specification for a benchmark that survives the frontier, and it is the exact inverse of a saturated public leaderboard: bounded, human-graded, and scraped. The industry is not fleeing to open math problems because it loves math. It is fleeing because those are the only rulers left with marks at the top.

## What to do Monday

Stop ranking models on a saturated public leaderboard. A two-point lead inside a five-point band is noise, and — on SWE-bench — half that band is mislabeled anyway.

If you need to tell two close models apart, size the experiment for it. To push the 95% interval below one point at p ≈ 0.9 you need about **3,500 clean items** — seven times SWE-bench Verified's 500. You almost certainly do not have that, which means you cannot resolve a two-point gap, which means you should stop pretending the leaderboard did it for you.

Build the private, post-cutoff, expert-curated eval on your own tasks — the discipline this desk has argued for since [the benchmark-is-not-the-capability](./2026-06-12-reading-a-coding-benchmark.md) dive — and size it to the gap you actually care about, not to a round number. Then measure the models on the axis that still has resolution: cost per solved task, tail latency, tool-call reliability. That is where the spread went when the benchmark ran out of it.

The number that decides your model choice is no longer on the leaderboard. It is the resolution of your own ruler — and if you have not measured that, you do not know whether your last ranking was a finding or a coin flip.
