# AI Levels the Field Where the Errors Are Cheap. It Widens It Where They Aren't.

*Deep dive · June Okafor (The Contrarian) · 2026-08-09 · This week the whole industry argued about whether AI makes coding skill matter more or less. Both sides are measuring the wrong thing.*

For four days straight the top of Hacker News has been one long argument about what a programmer is still for. "[LLMs reward expertise](https://www.seangoedecke.com/llms-reward-expertise/)" on Monday. "[Taste Is All That's Left](https://notashelf.dev/posts/taste-is-all-thats-left)" on Thursday. "[Why is everyone in tech so sad?](https://www.noemamag.com/why-is-everyone-in-tech-so-sad/)" — 953 points, 1,127 comments by this morning. And today, the rebuttal: "['Code was never the hard part' is an insult to all programmers](https://blog.senko.net/code-was-never-the-hard-part-is-an-insult-to-all-programmers)." The thread has turned into the week's real story, bigger than any release.

Underneath the mood there is a factual claim, and it is the one everyone repeats: **AI is a leveler.** It democratizes software. It closes the gap between the junior and the senior, makes execution nearly free, and leaves only judgment — taste — as the scarce thing. You have heard this since 2023. This week it just got two fresh coats of paint, one optimistic ("anyone can build now") and one elegiac ("so what's left of us is taste").

I want to take the leveling claim as seriously as it deserves, because it is not vibes. It has some of the best empirical support in the whole field. And then I want to show you why it is the wrong lesson for the people reading this — and why the residual skill it points to is not taste.

## The consensus, at full strength

Start with the two studies the optimists are right to cite.

The first is Brynjolfsson, Li, and Raymond's "[Generative AI at Work](https://academic.oup.com/qje/article/140/2/889/7990658)," now published in the *Quarterly Journal of Economics*. It is not a lab toy. It followed 5,179 customer-support agents through the staggered rollout of an LLM assistant. Productivity — issues resolved per hour — rose 14% on average. But the average hides the finding that matters here: the gain was **34% for the newest, lowest-skilled agents and roughly zero for the most experienced ones.** The tool worked by broadcasting what the best agents already knew to everyone else. It pulled the bottom of the distribution up toward the top. That is leveling, measured cleanly, at scale.

The second is Dell'Acqua and colleagues' "[Navigating the Jagged Technological Frontier](https://pubsonline.informs.org/doi/10.1287/orsc.2025.21838)," the Harvard–BCG experiment with 758 management consultants. On tasks inside the model's competence, AI users finished 12.2% more work, 25.1% faster, at markedly higher quality. And again the split by skill: consultants **below the average performance line improved 43%; those above it, 17%.** The gap between good and mediocre consultants shrank by more than half.

Two large, pre-registered, peer-reviewed field experiments. Both say the same thing. AI helps the weak more than the strong; it compresses the skill distribution. If you stop reading here, "AI democratizes knowledge work" is not hype. It is a result.

## Where the result stops holding

Now look at what those two tasks have in common, because it is the whole game.

A support ticket has a cheap, honest verifier: the customer's problem is solved or it isn't, and you find out in minutes. A consulting deliverable is graded loosely by a human rubric, and a mediocre slide that reads well is a *success* by that measure. In both settings the check on the work is fast, forgiving, and the cost of a wrong answer is bounded — one annoyed customer, one weak slide. The AI raises the floor because the floor is where a cheap verifier can see the improvement.

Run the identical experiment where those conditions fail, and the sign flips.

That experiment exists. In July, METR ran a randomized controlled trial with 16 experienced open-source developers working 246 real issues on **their own mature repositories** — code they knew cold, with tests, reviewers, and the silent-error surface of production software. The result: with AI tools the developers were **19% slower.** Not faster-and-they-didn't-notice — slower. And the calibration gap is the part to sit with. They had predicted a 24% speedup going in, and *after finishing the study* they still believed AI had sped them up by 20%. ([the study](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)).

Same technology, opposite outcome, and the opposite distributional story. The experts here were not leveled up by a generous floor; they were dragged down by the cost of checking a plausible-but-wrong suggestion against a codebase where wrong is expensive and quiet. The skill that separated them from a junior was never typing the code. It was knowing precisely what to ask for and catching the confident error before it shipped — and that is exactly the skill AI does not hand a novice.

## The variable nobody names

So which is it — leveler or concentrator? The honest answer is that the question has no general answer, and asking it that way is the mistake both camps make. AI's effect on the skill distribution is not a property of AI. It is a property of the **task's verifier.**

Sort any job by two numbers: how expensive it is to check whether the output is right, and how much a silent wrong answer costs you. Where the check is cheap and the downside is bounded — support replies, first-draft prose, boilerplate, throwaway scripts — AI levels, because a weak worker plus a fast check now reaches a good-enough answer, and that is what the studies measured. Where the check is expensive and errors are silent and costly — production code in a system you didn't write, a security boundary, a schema migration — AI concentrates, because the scarce input becomes the verification itself, and verification is the expert's job.

We've made this argument before from the model's side. Agents win at exactly the tasks that have a [cheap, faithful, runnable verifier](2026-07-24-verifier-asymmetry-check-vs-find.md) — a test suite, a compiler, a solved-or-not oracle — and flail where the only verifier is expensive human judgment. The distributional claim is the same law seen from the human side. When the verifier is cheap, the machine supplies it and the novice rides along. When the verifier is expensive, the machine can't supply it, so a human must, and the only human who can is the expert. Leveling and concentration are not two phenomena. They are one mechanism read at two verifier prices.

There is a nasty second-order term, too. The expert's verification skill is not fixed — it [decays with disuse](2026-07-22-human-in-the-loop-deskilled.md), and the model's confidence rises even as its accuracy on hard cases doesn't. So the very tasks where AI concentrates skill are the tasks where the concentrated skill is quietly rotting. That is not a reason to expect leveling. It is a reason to expect a widening gap between people who still practice the check and people who have outsourced it.

## What this says about the week's essays

With the variable in hand, the three essays sort themselves out.

Goedecke's "LLMs reward expertise" is *right*, and the field data even backs him where he offered none — but only in the expensive-verifier half of the world, and his own framing ("the human is the bottleneck… the difficult part is communicating exactly what kind of solution the human wants") undersells it. The bottleneck isn't communication in general. It's specification plus detection: saying precisely what correct means, and recognizing incorrect when the model hands it to you fluently. That is a narrower and more teachable thing than "expertise."

"Taste Is All That's Left" is half right in a way that matters. Yes, execution got cheap and judgment is what remains. But the essay names the residual "taste" — "the compressed, wordless verdict you reach faster than you can justify." In the domain most of us are paid for, the residual is not wordless and not aesthetic. It is *verification*: the failing test, the property, the review that catches the injected bug. It has right answers. You can measure whether you're good at it. Calling it taste makes it sound like an ineffable gift when it is a trainable, checkable engineering discipline — and the difference decides whether you can get better at it on purpose.

And today's rebuttal — "code was never the hard part" is an insult — is correct about the insult and imprecise about the fix. Its author is right that "[creating good code is a craft that requires skill, patience, attention to detail, experience and wisdom](https://blog.senko.net/code-was-never-the-hard-part-is-an-insult-to-all-programmers)." But the craft that survives isn't the typing, which genuinely did get commoditized. It's the part that decides whether the typed thing is right.

## So what, on Monday morning

If you manage engineers, the tempting move is to import the leveling result: hand juniors an agent and let them ship, because the studies say AI helps the inexperienced most. Do that only where the verifier is cheap — internal tools, well-tested modules, anything with a fast objective check and a bounded blast radius. In that zone it works and the research predicts it. Move the same policy into brownfield production code, where the verifier is a senior's attention and the errors are silent, and you have imported a result from a domain where it does not hold. The leveling doesn't transfer; the [jagged edge](https://pubsonline.informs.org/doi/10.1287/orsc.2025.21838) does — the same BCG study found that on tasks *outside* the frontier, AI users were 19 points *less* likely to be correct, and the ones who trusted it most did worst.

If you're the engineer: stop investing in output speed, which is now free, and start investing in the check, which is now the whole job. Learn to write the failing test first. Practice predicting the diff before you read it. Seed your own review with known-bad changes and measure how often you catch them — because a false-negative rate you have never measured is the real number, and it is drifting the wrong way. The market is about to pay a widening premium for people who can verify, and quietly stop paying for people who can only produce.

**What would change my mind.** Run METR's trial again — production issues, mature repos, real reviewers, correctness that matters — but stratified by developer skill, and show that AI *narrows* the defect-rate gap between juniors and seniors as models get more reliable. Show novices shipping senior-grade correctness in code they didn't write, because a machine supplied the verification a human used to. If the leveling result reproduces where the verifier is expensive, this whole argument is wrong and the democratization camp was right all along. So far, every time someone has measured that exact case, the gap has gone the other way.
