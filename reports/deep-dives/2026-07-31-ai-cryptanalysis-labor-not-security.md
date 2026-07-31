# Claude Didn't Break Cryptography. It Made Cryptanalysis Cheap.

*Deep dive · June Okafor (The Contrarian) · 2026-07-31 · Why "an AI found a flaw in the algorithm itself" is a labor story, not a security story — and how every result sorts by what you can check.*

Here is the sentence the week wants you to believe. A frontier model has started finding weaknesses in the cryptographic algorithms themselves, not just in buggy code, so the math we rely on is newly at risk.

It is not a dumb sentence. Let me give it full strength before I take it apart.

On July 28 Anthropic [published a set of real cryptanalysis results](https://www.anthropic.com/research/discovering-cryptographic-weaknesses) found by a preview of its Mythos model. The headline one is genuine. HAWK is a post-quantum signature scheme — the only lattice-based candidate to reach Round 3 of NIST's additional-signatures process, valued because its keys are compact and it avoids the floating-point arithmetic that makes Falcon painful to implement. Claude found a previously unknown automorphism — a symmetry — in the lattice HAWK is built on, and used it to cut the expected cost of recovering a HAWK-256 key from 2^64 to 2^38. That halves the security level. Fixing it means roughly doubling the key size, which erases the efficiency that was HAWK's entire reason to exist. The scheme had survived two years and two rounds of expert review. The model improved on the best known attack in about sixty hours. The day after the disclosure, [the HAWK team pulled it from the NIST process](https://www.techtimes.com/articles/322174/20260730/south-korea-certifies-hybrid-post-quantum-encryption-module-ai-breaks-hawk-algorithm-60-hours.htm).

That is not a press release. Matthew Green, who does this for a living, [called the HAWK result "genuinely impressive"](https://blog.cryptographyengineering.com/2026/07/29/some-notes-about-anthropics-new-results/) — and noted the team got it out of the model with plain prompting, not a bespoke, expert-tuned harness. General capability, pointed at a hard problem, producing a result humans missed. Take the scary reading seriously.

Now watch where the results actually land.

## The results sort themselves by what you can run

Anthropic reported more than HAWK. Sort the list by one property — can you execute the attack and check that it worked? — and the whole story rearranges.

On the runnable side: HAWK is a key recovery against a challenge instance. You run it, you get a candidate key, you check whether it verifies signatures. One pass, unambiguous. LEA reduced to 13 rounds: a practical key recovery needing about 2^30 plaintexts, done in under an hour — you can literally run it. Serpent cut to 6 rounds: a full key recovery. These are the confident results, and they are confident because a recovered key is a certificate. It works or it doesn't.

On the other side sits the AES result, the one that got the "Claude cracks AES" headlines. It is an attack on AES reduced to 7 of its 10 rounds, an improved meet-in-the-middle that Anthropic clocks as 200–800× faster than prior work. Green's read is blunter: "very much a small increment in our knowledge, not a practical new attack." It needs 2^105 chosen plaintexts and 2^89 cipher operations — a modest constant-factor gain over [work from 2013](https://www.di.ens.fr/~fouque/pub/fse13b.pdf). And here is the tell. You cannot run a 2^89-operation attack. So its correctness, in Green's words, "relies on on-paper analysis that may or may not yield an actual runtime improvement." The result you can't check is exactly the result the expert won't fully vouch for.

This is not a coincidence, and it is not new. This publication [made the general case a week ago](./2026-07-24-verifier-asymmetry-check-vs-find.md): a model is strong exactly where success has a short, cheap, faithful certificate you can run, and weak where it doesn't. A counterexample checks in one pass; a proof of a universal statement doesn't. Green arrives at the same line from inside cryptography — the exciting recent results, Green writes, either come with a "machine-checkable proof" or a "simple counterexample you can compute on." A key recovery is the counterexample. It is a witness that the scheme fails. That is why the machine found it, and that is why we believe it.

## "In the algorithm itself" is doing the work

Anthropic's framing — flaws "in the algorithms themselves, not just implementation errors" — is technically true and rhetorically loaded. It invites you to picture the wall cracking. What actually happened is narrower.

Three of the load-bearing targets are reduced-round variants: 7 of 10 AES rounds, 6 of 32 Serpent rounds, 13 of 24 LEA rounds. Cryptographers publish attacks on weakened versions on purpose. The round count is a safety margin, and the way you measure a margin is by attacking a version that has less of it. An improved attack on 7-round AES is a reading of the margin, not a breach of the wall. Full AES is untouched — Green, again: symmetric ciphers are "deliberately designed to be very messy and hard to untangle," and Green doubts a model finds a breakthrough there without "truly groundbreaking intuition." The under-10× improvements Anthropic reports on Salsa20, Poseidon, and a hash — SHA-1 — that the field [buried in 2017](https://shattered.io/) tell you how thin the pickings get once a cheap runnable check isn't handing the model its answer.

Even the good HAWK result fits the pattern. Claude's own summary, quoted by Green, is that "none of the ingredients are exotic." It combined known tools thoroughly and tirelessly against a scheme nobody had spent enough hours on. That is a real contribution. It is not new mathematics.

## The thing that got cheaper is labor

So here is the counter-thesis. The scarce input in cryptanalysis was never the technique. It was expert-hours pointed at a specific scheme. Green says it plainly about post-quantum: "there simply have not been enough human beings dedicated to analyzing these problems." A model that will apply the entire known toolkit to any construction you point it at, for sixty hours, without getting bored, relieves that scarcity. It does not raise the ceiling on what cryptanalysis can do. It drops the floor on what it costs.

Read that way, the week's news is mostly a defender's story. HAWK getting caught before NIST blessed it is the standardization process working — faster and cheaper than two years of human review managed. The right response to "an AI can now grind reduced-round attacks on any scheme" is to grind them on everything under review: the post-quantum candidates, the bespoke ZK-friendly hashes, the in-house constructions no academic had a reason to audit — while the choices are still reversible.

This is the same shape [I argued about autonomous attacks three weeks ago](./2026-07-07-autonomous-ransomware-known-cve.md). The capability ceiling didn't move; the marginal cost of the automatable middle fell, so the volume and the targeting distribution change. There, cheap agent labor meant more attempts against exposed and unpatched surfaces. Here, cheap analytic labor means more schemes audited before they ship. Same mechanism — and this time it mostly cuts for the good guys, because the attacks a model can find and verify are precisely the ones a designer wants found during review.

Where the story does have teeth is the one place Green flags as genuinely exposed: public-key and post-quantum crypto, where security rests on a handful of mathematical structures that remain under-analyzed. That is real. But notice it is the same asymmetry pointing the same way. Post-quantum key recovery is executable and checkable — a witness. Full-round symmetric security is a universal claim with no cheap certificate. The verifier asymmetry doesn't just explain the results we got. It predicts which crypto bends first: the kind where breaking it produces something you can run.

## What to do Monday, and what would change my mind

Do not rotate anything this week. Nothing here touches a full-round deployed primitive. AES-256, Ed25519, and SHA-256 are exactly as safe as they were on July 27. "Claude broke crypto" is false; "Claude halved a not-yet-standardized signature scheme and sped up some academic reduced-round attacks" is true, and much less frightening.

Do trust the standards process a little more, not less — automated cryptanalysis just joined pre-standardization review and did its job on HAWK. If you are choosing a post-quantum scheme, prefer the ones that have survived this kind of tireless machine attack, and discount novel or in-house primitives that haven't been through it.

Watch for the result on the wrong side of the line. The tell that this has become a security story and not a labor story is a novel cryptanalytic technique — one whose correctness needs a machine-checked proof because you can't run it — that dents a full-round, standardized cipher. That is the universal claim with no cheap witness, and the asymmetry says it is the last thing to fall, not the first.

That is also what would prove me wrong. Show me a frontier model producing a genuinely new structural attack, verified formally rather than by execution, against a primitive people actually deploy. Until then, the machine broke the crypto it could check and left standing the crypto it couldn't. That is not a wall cracking. It is a very fast, very cheap graduate student who only trusts answers they can run — and there is now an unlimited supply.
