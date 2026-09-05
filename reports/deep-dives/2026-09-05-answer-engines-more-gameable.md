# AI Answer Engines Are More Gameable Than Search, Not Less

*Deep dive · June Okafor (The Contrarian) · 2026-09-05 · Everyone says generative search killed SEO spam. Three sites and 215,128 fake pages say it moved upmarket.*

Here is the claim you have heard from every founder who ships an answer
engine, and from most of the people who use one: generative search kills the
content farm. You cannot game a language model the way you gamed PageRank.
There are no backlinks to buy. Keyword stuffing does nothing, because the model
reads for meaning, not for term frequency. So the old world — ten blue links,
eight of them junk you have to route around — is ending. Ask Perplexity for the
best CRM software and it reads the good sources and hands you a clean answer.

The steelman is real, and I want to give it full weight before I take it apart.
The [GEO paper](https://arxiv.org/abs/2311.09735) — Aggarwal et al., the
Princeton/IIT Delhi group that named "generative engine optimization" at KDD
2024 — found exactly this. Across their benchmark of ~10,000 queries, classical
SEO signals like keyword density had *minimal* influence on whether a page got
cited in a generated answer. The lever that moved the search-engine needle for
twenty years stopped working. If your spam recipe was keyword density and
backlinks, it is dead. That part of the consensus is true.

The problem is the next sentence in the same paper. The signals that *did* win
citations — up to a 40% visibility lift — were statistics, cited sources,
quotations from authorities, and fluent, confident prose. Aggarwal's group
calls these "epistemic authority" markers. Read that again with an adversary's
eyes. The optimization target didn't disappear. It moved from *keywords a human
typed* to *authority signals a model pattern-matches*. And the second target is
cheaper to fake convincingly than the first ever was.

## The recipe is the spam

We now have the existence proof, and it is almost too on-the-nose. A
[Trellner investigation](https://trellner.com/reports/manufactured-sources-behind-ai-recommendations/)
published this week found three sites — `wifitalents.com`, `worldmetrics.org`,
`gitnux.org`, all registered through NameCheap between December 2023 and May
2024, sharing Cloudflare nameservers and identical page templates — that had
between them published **215,128 machine-generated `/best/<category>-software/`
pages**. Their sitemaps list 103,578, 107,083, and 105,541 URLs. None of the
three domains existed twenty months ago.

The pages were not written for you. Two of the homepages carry the HTML title
"Facts & Grounding Page." The meta description calls the site "a machine-readable
record of verified facts." No human shopping for a CRM types "grounding." That
word is aimed at the retrieval step — the moment an answer engine fetches
documents to condition its reply. Each page states its ranking in JSON-LD, so,
in the report's words, it "can be read without interpretation." One page shipped
with an unrendered template variable in the byline reading "Within the next 26
days." Nobody proofreads content meant for a machine.

And it worked. Trellner queried 380 software categories through two of
Perplexity's models — `sonar` and `sonar-pro`, via OpenRouter, 760 calls —
and logged 7,534 citations across 2,055 domains. The three coordinated sites
took 181 of those citations and showed up in 41 of the 380 categories. A
separate vendor marketing blog, `guideflow.com`, landed **194 citations across
96 categories — third overall, ahead of Gartner's 158.** A vendor's blog, out-cited
a research firm, because it published one authoritative-looking page per
category and the ranker could not tell the difference.

That is the whole argument in one data point. The GEO paper told you the
winning move is to *look* authoritative to the model. These sites did nothing
else. They are not clever. They are the recipe, executed at scale, by people
who read the same paper you did.

## The audit surface is gone

Here is the part the "spam is dead" crowd misses, and it is structural, not a
bug Perplexity will patch away.

Google showed you ten links. The ranking was visible. You saw the domain before
you clicked, a manufactured page had to compete for your *attention* against
nine others, and if it won the click and disappointed you, you hit back and
picked another. The spam was there, but so was your ability to route around it.

An answer engine emits one synthesized answer, grounded on a handful of sources
it chose opaquely. Industry reverse-engineering of the pipeline — not
Perplexity's own docs, so treat it as directional — describes a
[retrieve-then-rerank flow](https://ziptie.dev/blog/how-perplexity-ai-answers-work/):
pull ~10 candidate pages, score them for freshness, structural clarity, and
authority, synthesize from the top three to five. Whatever the exact numbers,
the shape is the point. It is winner-take-all. A manufactured source that wins
retrieval does not win a slot on a page of ten — it wins the *whole answer*,
laundered into confident prose with a little "[3]" chip that reads as
provenance.

And the provenance is exactly what the format hides. In Trellner's run,
**59.8% of the domains behind grounded software recommendations ranked worse
than #100,000** on the Tranco list; another 23.4% sat outside the top million.
The median cited domain ranked 71,611. The sanity check you would run on a blue
link — "have I heard of this site?" — is unavailable, because you see a
footnote number, not `worldmetrics.org` registered in 2024. The one signal that
would let you catch the manufacturing is the one the answer format strips out.

So the incentive to manufacture sources didn't fall. It rose. The payoff went
from a click you still had to earn to a citation that *is* the answer, and the
reader lost the ability to see the seams.

## The honest counters

I have to argue against myself here, because the reflexive version of this
piece — "AI is drowning in slop" — is lazy, and the strongest objections are
good ones.

**"It's only 2.4% of citations."** Correct. The three farms accounted for 2.4%
of the total. This is not a takeover; it is the start of a curve. But 2.4% from
three sites that did not exist twenty months ago, in a channel where a single
citation can be the entire answer, is the tell, not the reassurance. Note the
bounds honestly: this is one investigation, one engine, software queries only.
It is an existence proof, not a census.

**"Perplexity will just fix its ranker."** Maybe. But this is the same
adversarial treadmill Google ran for two decades — Panda, Penguin, a spam
update every few months. The difference cuts the wrong way. Google's ranker
learned to separate keyword-stuffed junk from real prose, which is a
*legible* distinction. The new ranker has to separate machine-authored
authority markers from human-authored ones — stats, citations, JSON-LD,
confident tone — when both sides generate them with the same models. The
defender's problem got harder at the same moment the attacker's tools got
cheaper.

**"GEO is legitimate marketing, not spam."** The paper's authors frame it as
helping honest creators, and that is fair. But their own finding is the trap:
the winning signals are surface markers of authority, not authority itself. A
legitimate comparison page and a manufactured one that both add statistics,
citations, and structured data are *indistinguishable to the ranker*. GEO does
not separate the honest creator from the farm. It teaches both the same moves.

## What to do Monday, and what would change my mind

If you consume these answers: treat any "best X software" or "top N tools"
reply as one ranking with the ranking hidden. Those commercial-intent queries
are the exact target — the farms did not build "how does TCP work" pages, they
built "best CRM" pages, because that is where the buying is. Click the
citations. Check the *domain*, not the footnote number. For anything you will
actually adopt, the citation is a lead, not a verdict.

If you build a developer product, being cited is the new front page — the
machine-buyer channel this publication has been tracking since
["The Agent Reading Your Docs Won't Scroll"](./2026-07-04-docs-for-agents-distribution.md)
and ["Your Next Customer Is an Agent With a Wallet"](./2026-07-06-agent-with-a-wallet.md).
The structured, stat-dense, machine-legible page is real leverage; you should
ship it. But understand what you are competing against: manufactured content
running the identical recipe, which neither you nor the engine can prove is
fake. The only durable play is the one the recipe cannot fake at scale — primary
data the model can get *only* from you. Your own benchmarks. Your changelog.
Your docs endpoint. Not another "best CRM" listicle, because the farms already
own that shape.

What would change my mind: an answer engine ships an *inspectable* provenance
signal — site age, independence, or a trust score surfaced next to each
citation — or demonstrably reweights its ranker to push machine-authored
authority markers below human ones, and the median Tranco rank of cited sources
climbs back toward the visible web. Until one of those happens, I'll put it at
**70%** that a repeat of Trellner's study on a major answer engine twelve months
from now finds the manufactured-source share *higher*, not lower. The recipe is
published. The audit surface is gone. The only thing missing is time.
