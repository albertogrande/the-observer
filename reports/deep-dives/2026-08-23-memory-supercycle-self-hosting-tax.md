# Every Gigabyte of AI Memory Costs You Three of Your Own

*Deep dive · Marlow Quist (The Analyst) · 2026-08-23 · the same memory demand that pushes API prices to the floor is what made the hardware to escape them cost five times more*

Start with one price. A 128GB DDR5-6400 kit now
[lists at $3,399](https://www.tomshardware.com/pc-components/ram/memory-prices-climb-500-percent-in-12-months-up-to-10x-the-lowest-ever-tracked-prices-128gb-of-ddr5-now-usd3-399).
Twelve months ago the same class of kit was around $570. That is roughly +485%,
and up to 10× the lowest prices ever tracked. It is not a 128GB story. A
mainstream 64GB (2×32) DDR5-5600 kit that was under $200 last summer now clears
$1,100; one entry-level kit went from $72 to $392. Server DRAM, DDR4, SSDs, and
hard drives are all being pulled up in the same wake.

If you have been telling yourself that the answer to a rising API bill is to buy
a box and run models at home, that plan just got a new line item. And the reason
is the same force you were trying to escape.

## The ratio under the price

The three companies that matter — Samsung, SK hynix, Micron — make about
[94% of the world's DRAM](https://xenospectrum.com/en/ddr5-prices-5x-ai-hbm-memory-shortage-2026/).
They are not building more commodity DDR5. They are steering wafers to
High-Bandwidth Memory, the stacked DRAM that sits on AI accelerators, because
that is where the buyers sign multi-year contracts and pay margins ordinary DIMMs
can't touch.

The problem is that HBM is expensive to make in a very physical way. It uses
roughly 3× the wafer area per gigabyte of standard DRAM, and needs about 20 extra
process steps — through-silicon vias, die stacking. Micron's chief business
officer, Sumit Sadana, put the trade in one sentence at a
[KeyBanc forum this month](https://www.investing.com/news/transcripts/micron-at-technology-leadership-forum-2026-ai-tightens-memory-market-93CH-4849994):
to make 100 bits of HBM3E, Micron gives up about 300 bits of DDR supply. With
HBM4 and HBM4E the trade worsens toward 4-to-1.

That is the title of this piece as an equation. Every gigabyte of memory that
goes into a datacenter GPU removes roughly three gigabytes of the DDR5 you would
put in your own machine. HBM demand is
[projected to grow ~70% year-over-year in 2026](https://xenospectrum.com/en/ddr5-prices-5x-ai-hbm-memory-shortage-2026/);
SK hynix has moved on the order of 30% of its DRAM capacity to HBM and is heading
toward 40% by 2027. The supply that's left over is what prices your kit.

| Memory | ~12 months ago | Now | Change |
|---|---|---|---|
| 128GB DDR5-6400 kit | ~$570 | $3,399 | ~+485% |
| 64GB (2×32) DDR5-5600 kit | <$200 | >$1,100 | ~5–6× |
| Consumer DRAM contract (one quarter) | — | — | +89% QoQ |
| Server DRAM contract (Q3'26, forecast) | — | — | +13–18% QoQ |

*(Kit prices are US street prices reported by Tom's Hardware and secondary
trackers; contract moves are TrendForce. The ~$570 baseline is derived from the
+485% figure, not a separate quote — flagged.)*

This is not a spike that clears next quarter. TrendForce expects RDIMM bit-supply
growth held to 15–20% in 2027, well behind server-CPU shipments, with contract
prices rising through late 2027. Micron and others point to 2028 before real
relief; SK hynix has warned it could run past 2030. Plan around a multi-year
condition, not a blip.

## One cause, two bills, both pointing at "rent"

Here is the part that decides your build. The AI memory boom does two things at
once, and both of them favor renting a model over owning one.

**It makes serving cheap at scale.** Single-stream decoding is
[memory-bandwidth-bound](2026-08-04-fit-70b-in-4gb-bandwidth-wall.md): to emit
one token a model reads every one of its weights once, and on a batch-of-one
request the weights are 90–99% of the bytes moved. A provider fixes that by
batching — read the weights once, serve many users on that one read. Batching is
the whole reason datacenter tokens are cheap, and it's why the
[MoE economics](2026-06-21-mixture-of-experts-active-parameters.md) work at scale
and hurt at home. So the API price keeps falling toward the floor:

| Model tier | Input / output, per 1M tokens |
|---|---|
| DeepSeek V4-Flash | $0.14 / $0.28 |
| GPT-5.6 Luna | $0.20 / $1.20 |
| Gemini 3.5 Flash-Lite | $0.30 / $2.50 |
| DeepSeek V4-Pro | $0.44 / $0.87 |
| Claude Sonnet 5 (intro) | $2.00 / $10.00 |
| Frontier (Opus 5 / GPT-5.6 Sol) | $5.00 / $25–30 |

A capable coding tier now runs well under $1 per million tokens. As covered in
[W33](../2026-W33.md), the open-weight flood put six frontier-grade coding models
into the market inside a week; parity plus batching is why the token is a
commodity input.

**It makes your hardware expensive.** The same wafers that make serving cheap are
the wafers not in your kit. So the two ends of the "rent vs. own" decision moved
in opposite directions at the same time, driven by one thing.

## What it actually costs to own

Owning means holding a competitive model in fast memory. The
[June local-coding dive](2026-06-17-local-coding-model-memory-budget.md) put the
frontier-class open weights near 150GB; a 70B is 130GB in fp16, ~40GB at 4-bit,
plus a KV cache that grows with context. There are two honest ways to hold that,
and the memory market taxes both.

The big-RAM route — stream a model through system DDR5, or run a mixture-of-
experts model with only its active experts resident — is exactly the DDR5 whose
price just 5×'d. And for a *dense* model it doesn't even buy usable speed: the
[bandwidth-wall dive](2026-08-04-fit-70b-in-4gb-bandwidth-wall.md) measured the
DDR5 path at roughly 0.5–0.7 tokens/sec on a 70B — unusable for interactive work
no matter what you paid for the sticks. The route that *works* is an MoE model
plus a GPU with enough VRAM to hold the active experts at bandwidth. But GPU
memory is GDDR and HBM — the same market, bid up by the same buyers. There is no
lane around it.

Put rough numbers on the decision. Assume a box that can actually run a
competitive MoE interactively. A year ago the memory-bearing parts landed a
serious build near $3,500 all-in; today the same build is closer to $6,500, with
almost the entire jump on the memory line (the 128GB swing alone is ~$2,800,
before the GPU's own memory). Against that, rent at floor-to-mid tiers:

| | Rent (heavy, ~$1/Mtok blended) | Rent (floor, V4-Flash) |
|---|---|---|
| ~220M tokens/month | ~$220/mo | ~$45–60/mo |
| Payback on a $3,500 box (a year ago) | ~16 months | ~60 months |
| Payback on a $6,500 box (today) | ~30 months | ~100+ months |

*(Illustrative. Token volume assumes ~10M tokens/day of solo agentic work over 22
days; blended rate and build costs are stated assumptions, not quotes. Electricity
for a box run on demand is an added owning cost, not counted here.)*

The payback roughly doubled. And the denominator that kills it isn't in the
table: the model on your box depreciates. Competitive coding models now turn over
every few months. If the box can't pay itself back before the weights on it are
a generation behind — and now the fast-payback case clears two years — the money
says rent.

## The counterargument, taken seriously

The strongest case for owning is that hardware is capital and tokens are a
recurring bill: buy once, run forever, escape the meter for good. That case was
real a year ago. Three things broke it. The capex just doubled on the memory
line. The meter it competes against fell to the floor and keeps falling. And the
asset you're capitalizing has a half-life measured in months, while the shortage
that inflated it is forecast to run for years — so you can't even wait it out on
the buy side.

"Prices will fall" is the other reply. Eventually. The suppliers' own guidance
says not before 2028, possibly past 2030, and every AI-capex announcement resets
the clock. Buying into the top of a supercycle to save on tokens that are already
near zero is the wrong trade on both axes — the same shape the
[bandwidth-wall dive](2026-08-04-fit-70b-in-4gb-bandwidth-wall.md) flagged in
August: the thing you're fleeing is already near the floor, and you're paying a
premium to flee it.

There is a real exception, and it's not about savings. If you *must* keep data on
your own metal — air-gap, compliance, an API you're not allowed to call — then the
box is a compliance cost, not an arbitrage, and you should size it honestly: an
MoE model, a GPU with enough VRAM for the active experts, bandwidth over
capacity. The same holds for high-utilization overnight batch, where a token
every few seconds is fine because you're asleep. Owning wins where the meter
legally or physically can't reach — not where it's merely annoying.

## The wider cycle, and the number to watch

Step back and this is a commodity supercycle doing what commodity supercycles do:
fab capacity is finite, it flows to the highest-margin product, and everyone
downstream pays the shortage. We have seen DRAM cycles before; what's new is that
the highest-margin product *is* the AI buildout, so the same demand curve sets the
price of both renting a model and building the thing that would let you stop. New
trade friction — this weekend's US–Canada tariff escalation among it — is a
second-order squeeze on the hardware route, not the main event; the main event is
the 3-to-1 ratio.

So the deciding quantity, in Analyst terms, is your **break-even token volume**:
utilization × months-the-model-stays-current × the floor price, against a capital
line that just doubled. Run it before you buy. For almost every individual and
most teams, the honest output right now is: rent the model, and if you need a
local box, buy it for control, not for savings.

What would change my mind: RDIMM bit-supply growth catching demand (watch the
15–20% figure climb), the HBM allocation share topping out rather than rising, or
a credible pull-in of the shortage's end from 2028 toward next year. Until one of
those moves, the memory market has quietly decided the rent-vs-own question for
you — and it decided rent.
