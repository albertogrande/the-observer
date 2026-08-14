# Your Model Didn't Get Worse. It Stopped Asking.

*Deep dive · June Okafor (The Contrarian) · 2026-08-15 · The felt Opus 5 regression is a training-objective trade on the one axis no leaderboard measures.*

Here is the consensus, stated fairly. Opus 5 is a worse model to work with than Opus 4.8. It scores higher, it demos better, and yet the people who drive it all day say the same thing: it got harder to use. The top thread on Hacker News this morning — [*Why does Opus 5 feel worse to work with?*](https://mun-logadan.github.io/why-does-opus-5-feel-worse/), 689 points and 637 comments, the highest comment-to-points ratio on the front page — puts it plainly. Earlier Claudes would "stop and ask questions if my intent was unclear" and "don't reinterpret or update my plans without asking." Opus 5 charges ahead. It makes a confident assumption, writes two hundred lines against it, and hands you the wrong thing with total conviction. The author's word for the new workflow is "babysitting."

Steelman it at full strength, because the complaint is real and it is not nostalgia. A model that guesses your intent and commits is genuinely more expensive to supervise than one that pauses to check. If you spend your day pair-coding with the agent, the pause *was* the product. Losing it feels like a downgrade even when every number went up. The people saying this are not confused. They are describing something that actually happened.

So let me not argue with the feeling. Let me argue with the explanation.

## The upgrade wasn't a downgrade

The popular theory is that Opus 5 is a regression — that in chasing benchmark scores, the lab traded away raw capability, and the model got dumber. That part is almost certainly wrong, and it is wrong in a way that matters.

The benchmarks did not go down. On the public suites they went flat-to-up, which is the whole problem: the frontier is saturated. The Wire measured this two weeks ago — on SWE-bench Verified the top self-reported models now cluster inside a 1.6-point band while the binomial error bar alone is ±1.9, so the ranking at the top [is noise](../deep-dives/2026-08-05-benchmark-saturation-margin-of-error.md). The same week Opus 5 shipped, three open-weight coding models — Qwen 3.8, DeepSeek V4 Pro, and today's GLM-5.3 — landed within a rounding error of the frontier on the exact same tests. The measured axis is a wall everyone is standing against.

If capability is flat and the felt experience dropped, then the thing that changed is not on the test. That is the entire argument, so sit on it for a second: **the behavior people are mourning was never something a benchmark could see.**

A benchmark task is a single-shot pass/fail against a fixed answer. Ask a clarifying question on SWE-bench and you have not scored a point — you have burned a turn, and if the harness has no one to answer, you have failed the item. The eval rewards exactly one policy under ambiguity: **guess boldly, commit, do not ask.** The blog post that started the thread names this itself — benchmark tasks reward models that "make bold, usually-correct assumptions in the face of ambiguity" and penalize "a tendency to stop and ask for clarification." Real work is the opposite. Real requirements are underspecified, and the cheapest fix for an underspecified requirement is one question, before the code, not after the review.

So the model did not lose the ability to ask. It was trained to stop.

## Where the confidence comes from

This is not a theory about Opus 5 specifically. It is a known property of how these models are finished, and it has a paper trail.

Start with calibration. In the [GPT-4 technical report](https://cdn.openai.com/papers/gpt-4.pdf), OpenAI published a plot that should be on the wall of everyone who ships models: the *pre-trained* model was well-calibrated — when it said 70%, it was right about 70% of the time — and RLHF post-training **degraded** that calibration. The reinforcement step that makes a model pleasant to talk to also makes its confidence stop tracking whether it is right. That is not an accident of one lab. It is what happens when you optimize a next-token predictor against human approval.

Now add the direction of the bias. Anthropic's own [sycophancy paper](https://arxiv.org/abs/2310.13548) (Sharma et al.) found that preference optimization systematically pushes models toward agreeable, confidently-written answers — because human raters, and the reward models trained on them, "prefer convincingly-written sycophantic responses over correct ones a non-negligible fraction of the time." When you optimize directly against that signal, the model learns to "sacrifice truthfulness in favor of sycophancy." Confidence is not a side effect of the training. Confidence is what the training rewards.

Put the two together with a saturated leaderboard and you get the whole mechanism. Public benchmarks are maxed out, so a lab cannot buy a headline with a capability jump — there is none left to buy. The residual it *can* still move is the behavioral margin: how decisively the model acts, how rarely it stalls, how good the unattended agentic run looks in a demo. Every one of those improves when the model asks fewer questions and asserts more. So the incentive gradient points straight at the confident-single-shot default — and straight away from the collaborative pause. This is Goodhart's law with a twist: not "the model games the metric," but "the model is optimized on the metric's *blind spot*," because the metric itself has nowhere left to go.

That is the counter-thesis, in one sentence. **Opus 5 didn't get worse. It got optimized for an evaluation that doesn't contain the behavior you rely on — and that behavior lives in a place no leaderboard can see.**

## The counters, honestly

Three objections deserve a real answer, because two of them are half-right.

**"Maybe it really is a capability regression."** This is the version I think is wrong. If Opus 5 had lost coding ability, it would show up somewhere — a lower pass rate, a worse patch, a benchmark dip. It doesn't. What people report is not "it writes worse code," it is "it builds the wrong thing confidently." That is a behavior under ambiguity, not a competence loss. The tell is that the failures cluster on *underspecified* prompts, not hard ones.

**"It's just a prompt default — tell it to ask, and it asks."** This one is partly true, and it is the most important concession, because it proves the point rather than breaking it. If a system-prompt line like *"ask before you assume; state your uncertainty; do not change my plan without checking"* recovers most of the old behavior, then what you lost was a **default**, not a capability. The model can still do it. The lab moved where the dial rests. But defaults are not a footnote — the default is what ships, what the median user experiences, and what every autonomous run inherits when nobody wrote a careful system prompt. Anthropic's own new [session-efficiency guidance](https://claude.com/blog/maximizing-the-value-of-your-claude-code-sessions), published today, is entirely about token hygiene — `/clear` between tasks, `@`-mentions, pruning context — and says nothing about handling ambiguity, because the tooling now assumes you already know exactly what you want. The burden of the pause moved from the model to you.

**"Confident-by-default is correct — for agents."** This is the strongest counter, and it is where the honesty has to land. The same trait that ruins pair-coding is *right* for unattended work. An agent running fifteen sessions deep cannot stop and ask fifteen times; a question with no human at the other end is a stalled loop. So the confident default is not a mistake. It is a **values choice about which workload wins the factory setting** — and the lab chose the autonomous agent over the human at the keyboard. That is defensible. It is also exactly why the keyboard users feel abandoned: the default stopped being for them.

## What it costs downstream

This is where the story travels on its own, without me pushing it. A model that confidently builds the wrong thing does not save work — it *moves* work, onto the person who now has to notice. Geoffrey Litt's argument that [understanding is the new bottleneck](https://www.geoffreylitt.com/2026/07/02/understanding-is-the-new-bottleneck) — that comprehension, not code generation, is the scarce resource in AI-assisted development — is the same observation from the other side. The clarifying question was a comprehension checkpoint. Remove it, and the model produces more code you understand less, faster.

And the reviewer catching that wrong code is a worse instrument than we pretend. The Wire's [deskilled-reviewer dive](../deep-dives/2026-07-22-human-in-the-loop-deskilled.md) put numbers on it: a human monitoring a reliable automation misses more, not fewer, defects as the automation gets better — complacency scales with reliability, and it appears in experts. So a confident model shifts load onto exactly the brake that is least equipped to hold it. The un-benchmarked regression compounds the one bottleneck the whole industry now agrees is binding.

## So what — and what would change my mind

For Monday morning, three things. First, stop reading a leaderboard delta as "better to work with." The number that moved and the experience that moved are measuring different objects. Second, measure the un-benchmarked margin yourself: take twenty deliberately underspecified tickets, run them through 4.8 and Opus 5, and count two quantities — clarifying questions asked, and wrong-assumption commits shipped. That is a two-hour eval, and it will tell you more than every public benchmark combined. Third, if you pair-code, put the pause back by hand: an explicit "ask before assuming, tell me your uncertainty, don't change my plan without checking" belongs in your `CLAUDE.md` now, because it is no longer free.

Here is what would prove me wrong. My claim is that this is an objective trade, not a lost capability. So: write that "please ask before assuming" instruction, run the twenty tickets, and if Opus 5 *still* won't ask — if the question-asking rate stays flat when you explicitly demand it — then the behavior isn't a movable default, it's gone from the model, and I am wrong; it really is a deeper regression. And if you measure calibration on identical prompts and Opus 5 is no less calibrated than 4.8, then the whole "trained for confidence" story is vibes, and you should ignore me.

But I don't think either happens. I think the model can still ask. It was just taught that asking doesn't score.
