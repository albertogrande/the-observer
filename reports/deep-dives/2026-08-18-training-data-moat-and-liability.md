# The Moat Is Made of the Thing They Can Never Show You

*Deep dive · June Okafor (The Contrarian) · 2026-08-18 · Everyone says training data is the moat. It is also the liability the moat is built from — which is why no frontier lab will ever open its corpus.*

Here is the sentence everyone repeats. **Data is the moat.** The weights are a
commodity now (we have said so here, most of the summer). Open models match the
frontier on the benchmarks. The one thing a rival can't copy is the training
corpus — the trillions of clean, curated, deduplicated tokens a lab spent years
and hundreds of millions of dollars assembling. That's the asset. That's why the
labs win.

I believe the first half of that. I want to break the second half.

The steelman is strong, so take it at full strength. Scaling is data-hungry, and
the supply of high-quality text is finite — the "data wall" is real, not a
talking point. Preference data is a genuine moat: every accept/reject on a real
coding task is a labeled pair you can't scrape, and we argued exactly that in the
[accept-button dive](./2026-07-10-accept-button-is-the-moat.md). Assembling a
clean corpus is expensive, boring, and hard to reproduce. And the optimists have
something better than vibes — they have a *ruling*. In June 2025 Judge William
Alsup found that training Claude on books was "exceedingly transformative" and
therefore [fair use](https://www.authorsalliance.org/2025/06/24/anthropic-wins-on-fair-use-for-training-its-llms-loses-on-building-a-central-library-of-pirated-books/).
Training is legal. The moat is legal. Case closed.

Except that was only half the ruling, and the labs' own behavior tells you which
half they're afraid of.

## The half nobody quotes

Alsup split the case cleanly. Training on lawfully acquired books: fair use.
*Acquiring* the books by downloading at least five million from Library Genesis
and two million from the Pirate Library Mirror: not fair use, not excused,
straight to a damages trial. The transformation defense protects what you *did*
with the data. It does nothing for how you *got* it.

Anthropic did not go to that trial. It [settled for about $1.5 billion](https://authorsguild.org/news/court-grants-final-approval-anthropic-copyright-settlement/) —
roughly $3,000 per work across an estimated 500,000 books, the largest copyright
recovery in United States history, final approval in July 2026. Read that number
against the meter it settled. The statutory ceiling for willful infringement is
$150,000 per work. Take just the ~482,460 works in the certified class, multiply
by that ceiling, and you get about $72 billion. Run the full seven-million-plus
pirated copies through the same arithmetic and the theoretical exposure climbs
into the hundreds of billions — a number larger than the company. The $1.5
billion wasn't a fine. It was Anthropic buying its way off a number that could
have exceeded its valuation, for a corpus that had already been trained on, in a
model that was already shipped and already earning.

That is the shape of the thing. The asset was consumed years before the invoice
arrived. The moat and the liability are the same bytes.

## "The model doesn't store the books" is false, and they know it

The comfortable technical defense is that a model is not a copy. It's a
compression of patterns; the training text is gone; you can't get it back. As an
absolute claim, that is measurably wrong, and the labs measure it more carefully
than anyone.

Carlini and colleagues quantified it in
["Quantifying Memorization Across Neural Language Models"](https://arxiv.org/abs/2202.07646).
Memorization is not zero and not random. It grows log-linearly along three axes:
model capacity, prompt-context length, and — the one that matters here — **the
number of times an example was duplicated in training.** A six-billion-parameter
GPT-J memorizes at least 1% of its training set verbatim, and newer probabilistic
extraction pulls out several times more than naive greedy sampling finds. The
bigger the model and the more a passage repeats in the corpus, the more reliably
it comes back out word for word.

Now connect the wires. Pirated book collections are heavily duplicated — the same
bestseller sits in a dozen shadow-library dumps. Duplication is exactly the axis
that maximizes memorization. So the highest-liability text in your corpus is also
the text your model is most likely to regurgitate on demand, and a regurgitation
is not an abstract compression argument — it is a copy the plaintiff puts on a
screen next to the original. This is why every serious lab spends real engineering
on deduplication before pretraining. It is framed as a quality and
compute-efficiency measure. It is also litigation hygiene. You dedupe so the model
memorizes less of the thing you can't prove you were allowed to have.

## Why the corpus is the one asset they will never open

Here's the counter-thesis, stated plainly: **data isn't the moat and, separately,
a liability. The moat is made of the liability.** And that changes what the
secrecy means.

We usually read a lab's silence about its training data as competitive — of
course they won't hand rivals the recipe. But the recipe isn't the sensitive part.
The *manifest* is. A full, work-level list of your sources is a plaintiff's
membership roster: it tells every rights-holder whether they're in, and it hands a
class-action lawyer the class. Disclosure is discovery. That's why "open source
AI" keeps arriving without the data — as we covered when the
[OSI's definition](./2026-06-16-open-weights-is-not-open-source.md) landed, OSAID
1.0 asks for "data information," and no frontier model meets it. The weights open;
the corpus never does. The reason isn't only that the corpus is valuable. It's
that the corpus is evidence.

Which brings us to the number in this week's news. Anthropic's IPO, per Reuters,
[hinges on a 2028 revenue forecast of $190–200 billion](https://www.reuters.com/business/anthropic-ipo-valuation-hinges-190-200-billion-2028-revenue-forecast-sources-say-2026-08-15/) —
against a run-rate that was about $9 billion at the end of 2025 and roughly $47
billion by May. Bankers are pricing the company on a revenue multiple two years
out. A revenue multiple prices the corpus as a pure asset: it shows up in the
tokens sold, never in a liability line. But the corpus already generated a $1.5
billion cash event once, on one category of one medium — books — with music,
news, images, and code corpora still working through the courts. The tail that a
revenue multiple assumes never lands has already landed once. It is not priced
into the multiple; by construction, it can't be.

## The strongest thing said against me

The honest counter is that Alsup handed the labs the ruling that matters. Training
is fair use. Acquisition is a solved, one-time cleanup cost: buy the books
legally, or license the catalog, and the liability evaporates going forward. On
this view the $1.5 billion is a closing entry, not a running meter — the price of
sins committed before the law was clear, now paid.

Partly true, and it's why I'd bound my claim rather than shout it. The
forward-looking labs really are moving to licensed and owned data, and a
purpose-clean corpus does close the acquisition hole. But two things keep the
liability alive. First, the models earning revenue *today* were trained on the
old corpora; you cannot un-train a shipped model, so the exposure rides with every
weight already in production. Second, "licensed" is a per-medium, per-jurisdiction
grind, not a switch — a book deal doesn't cover the lyrics, the lyrics deal
doesn't cover the news, and the news deal doesn't cover the GitHub repos whose
licenses forbid exactly this. The acquisition liability isn't erased. It's
amortized across a decade of separate fights, each one its own $1.5 billion
question.

## So what, at the keyboard

If you ship on top of these models, the corpus problem is upstream of you, but its
handling is a control you can read.

- **Read "we don't train on your data" as a legal instrument, not a courtesy.**
  Zero-data-retention and Privacy Mode exist to keep your inputs out of the next
  training run — which protects the vendor's liability surface first and yours
  downstream. That's a reason to trust it more, not less. Turn it on for anything
  proprietary and treat it as the default.

- **The provenance question isn't "is the model good." It's "what's the
  indemnity, and what does it actually cover."** As the
  [provenance dive](./2026-08-10-provenance-is-the-product.md) laid out, vendor IP
  indemnities defend *you* against a third party's claim under conditions; they are
  not a transferable clean-provenance warranty you can pass down a contribution
  agreement. If your project stakes an IP warranty on every commit, the model's
  training data is your problem the moment its output regurgitates something.

- **A model's silence about its corpus is data.** It is the shape of the
  liability, not modesty. The labs that say the least about their sources are
  telling you where the exposure is.

**What would change my mind.** Two things, and I put both below 30% before the end
of 2027. One: a frontier lab ships a genuinely top-tier model on a fully
disclosed, fully licensed corpus — publishes the manifest — and does *not* draw a
wave of suits, proving the corpus can be shown without being evidence. Two: an
appeals court squarely holds that acquisition via pirated copies is itself fair
use, collapsing the half of the ruling the labs actually settled. Absent either,
expect the pattern to hold: the weights keep opening, the benchmarks keep
converging, and the one asset that stays locked in every lab's vault is the one
they can never afford to let you read.

The moat is real. It's just that a moat you can never drain, inspect, or show to
your own auditor isn't only protecting you. It's also the thing that can sink you,
and it's priced at zero in every number this industry is about to go public on.
