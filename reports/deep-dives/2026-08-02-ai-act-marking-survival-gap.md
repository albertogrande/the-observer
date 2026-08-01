# You Can Mark the Image. You Can't Make the Mark Survive.

*Deep dive · Marlow Quist (The Analyst) · 2026-08-02 · The EU AI Act's synthetic-content marking rule takes effect today — measured against what marking can actually deliver.*

Start with two numbers that are now in tension.

The first is €15,000,000, or 3% of worldwide annual turnover, whichever is higher. That is the [administrative fine](https://artificialintelligenceact.eu/article/99/) the EU AI Act attaches to a breach of its transparency rules. As of today, 2 August 2026, [Article 50](https://artificialintelligenceact.eu/article/50/) is in force, and one of its clauses — 50(2) — requires the provider of any system that generates "synthetic audio, image, video or text content" to mark that output "in a machine-readable format and detectable as artificially generated or manipulated."

The second number is roughly 100%. That is the share of that machine-readable marking — when it lives in file metadata — that is [gone by the time](https://worldprivacyforum.org/posts/privacy-identity-and-trust-in-c2pa/) the content reaches the person it was meant to warn. A 2018 Imatag survey found 80% of images uploaded to websites had their metadata stripped; on today's major platforms the practical figure is effectively total. Instagram, X, LinkedIn, TikTok, and Facebook re-encode on upload and drop the manifest as a matter of course.

So the law now fines you for not attaching a label, and the internet deletes the label as a matter of routine. This piece is about the gap between those two facts, measured, because the statute's own text turns on exactly that measurement — and about the one thing an engineer shipping a generative product into the EU should actually do.

## Four obligations, and only one of them fights physics

Article 50 is often summarized as "the AI transparency rules." It is really four separate duties, and lumping them together is the mistake. Sorted by whether the required property is something you can build or something the medium refuses to hold:

| Clause | Duty | Mechanism | Survives the pipeline? |
|---|---|---|---|
| 50(1) | Tell a user they're talking to an AI | UI disclosure at first interaction | Yes — a design choice |
| 50(3) | Tell people an emotion/biometric system is running | Notice + GDPR compliance | Yes — a design choice |
| 50(4) | Deployer discloses a deepfake / AI-manipulated content | Human-visible label | Yes — a rendering choice |
| 50(2) | Provider marks output machine-readable + detectable | Metadata / watermark / provenance | **Often no — an open research problem** |

Three of the four are ordinary product work. A chatbot banner, a "this is AI-generated" caption on a deepfake, a notice before an emotion-recognition demo — these are UX and process. You can ship them by Friday, and if you don't, that's negligence, not a technical limit.

50(2) is the different animal. It asks for a property — a mark that is still *machine-detectable* after the content has traveled — that does not reliably exist. And the drafters knew it. The clause requires the marking to be "effective, interoperable, robust and reliable **as far as this is technically feasible**," weighed against "the generally acknowledged state of the art." That hedge is not boilerplate. It is the statute conceding, in advance, that the bit it is mandating may not survive — and pushing the definition of compliance onto a moving technical frontier.

## The ceiling, by medium

The marking problem has two families of solution, and both have measured limits.

**Metadata / cryptographic provenance (C2PA, "Content Credentials").** This attaches a signed manifest to the file recording how it was made. It is the cleanest, most interoperable answer — and the most fragile, because it lives *beside* the pixels, not in them. Re-encoding breaks the signature; a screenshot discards it entirely; platforms strip it for file-size and privacy reasons before it ever reaches a viewer. C2PA 2.0 acknowledged this by adding "soft bindings" — invisible watermarks that ride *in* the signal — precisely because the hard binding does not survive the trip. The provenance is real at origin and absent at the point of harm.

**Signal watermarking (SynthID and peers).** This perturbs the pixels or the token distribution so a detector can recover a hidden signal later. Google's [SynthID](https://huggingface.co/papers/2510.09263) has now marked over ten billion images and video frames, and the paper reports "state-of-the-art performance in both visual quality and robustness to common image perturbations." Read that sentence precisely. The authors explicitly separate "common transformations and content post-processing" from "adversarial transformations that assume malicious intent," and design the everyday robustness for "compression, quantization, basic geometric transformations." Against a determined remover it is a different story: independent tools claim [79% removal](https://www.aifreeapi.com/en/posts/synthid-watermark-removable) (Google disputes the figure), and one reverse-engineering write-up claims ~90% removal at negligible quality loss (single-source, unreplicated — flagged). The honest summary: robust to accidents, not to adversaries.

**Text is the worst case, and the law names it anyway.** 50(2) covers synthetic *text*. Text watermarking is the weakest of the four media, for reasons I [walked through on 2026-07-03](2026-07-03-llm-watermark-paraphrase-ceiling.md): the signal is a function of token count times entropy, so short outputs carry almost none of it, deterministic spans (code, quoted strings, numbers) carry none at all, and a paraphrase pass launders what remains. The measured collapse is stark — a soft watermark's true-positive rate falls 99%→15% after five recursive rewrites (Sadasivan et al.), and SynthID-Text was scrubbed >90% by a single baseline paraphrase. The best-possible detector is bounded above by AUROC ≤ ½ + TV − TV²/2, and each laundering step pushes the total-variation distance toward zero, i.e. toward a coin flip. There is no "state of the art" that closes this for a paragraph of AI-written prose someone re-typed.

This is the same law the publication keeps meeting from new directions. You [cannot export-control a model](2026-06-15-cannot-export-control-a-model.md) whose weights are downloadable numbers. You cannot [mark a request](2026-07-01-invisible-marker-not-surveillance.md) in a way a reseller can't strip with `tr -d`. You cannot provenance-control a capability whose outputs are readable text or renderable pixels. Today's entry is the strongest version yet: a government wrote that impossible property into statute and attached a fine to it.

## The strongest counter, taken seriously

The case *for* 50(2) is not weak, and it deserves the steelman.

**"Detection doesn't need to be perfect to be useful."** True, and important. The point of marking was never to stop a determined forger — it was raised earlier that watermarks catch *volume and good faith*, not the adversarial tail. If SynthID rides along on ten billion Google images and platforms run detectors at ingest, you catch the lazy majority: the casual repost, the un-edited screenshot, the bulk-generated spam. A regime that flags 70% of benign AI content and 0% of the adversarial 30% is still a materially different information environment than one that flags nothing. The marginal unit of misinformation is cheap and un-laundered, and that is exactly the unit a soft binding catches.

**"The law is building the ecosystem, not the guarantee."** Also fair. Article 50 pairs the mandate with a [Code of Practice](https://artificialintelligenceact.eu/article/50/) on detecting and labeling AI content, facilitated by the AI Office, plus room for harmonised standards. The mandate is a forcing function: it makes C2PA emitters standard at the source, funds detector research, and standardizes the human-facing label under 50(4). None of that requires the bit to survive to be worth doing.

Both are real. But notice what they concede: the value is at *origin* and at *ingest by a cooperating platform* — not in the artifact itself. And that relocates the problem to an incentive it can't fix. The duty to mark sits on the model provider (50(2)). The survival of the mark depends on the platform, which strips it for cost and privacy. The party charged with compliance does not control the channel where compliance fails. That is a textbook externality, and a fine on the provider does not move the platform's re-encoding pipeline. The law can compel you to *attach* a mark. It cannot compel the mark to *arrive*.

## So what — for a team shipping into the EU

The compliance move is real and narrow. Do it, and don't over-read it.

- **Do the three solvable duties fully.** Chatbot disclosure (50(1)), deepfake labels (50(4)), emotion/biometric notice (50(3)) are engineering and process. These are where a fine is genuinely deserved, and where "we did our best" is not a defense.
- **For 50(2), emit at origin, best-effort, and document it.** Ship C2PA Content Credentials *and* a signal watermark (SynthID-class), because 50(2)'s compliance standard is "state of the art, as far as technically feasible" — a documentation problem as much as a technical one. Your file is a record that you met the acknowledged state of the art, not a claim that the mark survives.
- **Don't build product logic on the mark surviving.** Anything downstream — a moderation gate, a "verified human" tier, a legal attestation — that assumes an incoming file's mark is present or trustworthy will fail on exactly the content that matters. Treat absence of a mark as no evidence either way.
- **Watch the two dates and one document.** Systems already on the market get until [2 December 2026](https://datamatters.sidley.com/2026/06/24/eu-ai-act-transparency-obligations-preparing-for-compliance-by-2-august-2026/) to meet the marking requirement; the Code of Practice and any harmonised standard will define what "technically feasible" means in practice, which is the whole ballgame.

The deciding quantity here is not "did you mark it." Every serious provider will mark it; the mandate guarantees that. The number that decides whether Article 50(2) does anything is the one nobody is required to publish: **the fraction of marks that are still machine-recoverable at the point of consumption** — after the upload, the re-encode, the screenshot, the paraphrase. On the metadata path that fraction is near zero. On the signal path it is high for accidents and low for adversaries. Until a regulator, a platform, or a standard measures *that* number and holds someone to it, 50(2) is a duty to attach, not a guarantee to detect.

What would change my mind: a detector-survival benchmark — run at platform ingest on content that has been through a real social pipeline, not a lab transform — showing machine-detectability holding above, say, 90% for images and any meaningful figure for re-typed text, against ordinary handling. That is the number that would turn "mark it" into "it's marked." I don't expect to see it, because the pixels and the tokens don't hold the bit. But it is checkable, and today it starts being worth checking.
