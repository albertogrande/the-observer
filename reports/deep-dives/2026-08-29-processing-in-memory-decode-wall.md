# The Weights Move to the Multiplier. Samsung Moved the Multiplier to the Weights.

*Deep dive · Marlow Quist (The Analyst) · 2026-08-29 · Processing-in-memory finally shipped a measured part. The question is whether it beats the bandwidth wall before software and on-die SRAM close it first.*

Start with the number that has embarrassed every inference accelerator for a
decade. When an H100 generates one token, its arithmetic intensity — math
operations per byte pulled from memory — is about **1 op/byte**. The chip's
roofline ridge point, where compute and bandwidth balance, sits near
**295 ops/byte** ([the H100 does 989 FP16 TFLOPS against 3.35 TB/s of
HBM3](https://www.abhik.ai/concepts/gpu-computing/hbm-memory)). So batch-1
decode runs about **1/295 ≈ 0.34%** of the way up the compute the card
advertises. The other 99.7% of the multiplier waits on the memory bus.

We have said the *why* twice this summer. Batch-1 decode reads every weight
from memory, multiplies it once, and throws it away, so single-stream
tokens/sec ≈ effective bandwidth ÷ model bytes — [the bandwidth wall, 08-04
](./2026-08-04-fit-70b-in-4gb-bandwidth-wall.md). And memory is now the
expensive part of the machine: [DRAM up ~485% in a year, HBM eating the
wafers, 08-23](./2026-08-23-ai-memory-supercycle-self-hosting-tax.md).

This month a third piece landed. At Hot Chips 2026, Samsung showed
[LPDDR5X-PIM](https://www.tomshardware.com/pc-components/dram/hot-chips-2026-samsung-makes-lpddr5x-smart-with-logic-unit-in-memory-lpddr5x-pim-is-3-01x-faster-than-lpddr5x-in-ai-inference-with-8x-the-bandwidth):
commodity phone memory with a multiplier built into each DRAM bank. Samsung
built the part, ran Llama-3.1-8B on it, and measured **3.01× the tokens per
second** of the same memory without the logic. Processing-in-memory has been
a slide for fifteen years. Now there is a measured part. Here is the
arithmetic on whether it matters.

## The one thing PIM changes

Memory has always had two bandwidths, and the specs only quote one. Inside a
DRAM die, the banks read out at enormous width. Then everything funnels
through the package pins to the bus, and the pins are the bottleneck the
datasheet reports. PIM puts a small compute unit *inside*, at the bank
boundary, so a matrix-vector multiply runs on the internal bandwidth and only
the tiny result vector crosses the pins.

The gap is not marginal:

| Where the bytes move | Bandwidth | Notes |
|---|---:|---|
| Conventional LPDDR5X (pins/bus) | 76.8 GB/s | what the datasheet quotes |
| **LPDDR5X-PIM (bank-internal)** | **614 GB/s** | 8× the pin bandwidth, same 561-ball package |
| SK hynix GDDR6-AiM (bank-internal) | 512 GB/s | per-bank 1 GHz unit, GDDR6 at 16 Gbps |
| HBM3 stack ×5 (an H100) | 3,350 GB/s | the datacenter answer, at datacenter cost/power |
| On-die SRAM (Cerebras WSE) | ~1,000,000 GB/s | vendor figure; no DRAM at all |

Sources: [Samsung / Tom's Hardware](https://www.tomshardware.com/pc-components/dram/hot-chips-2026-samsung-makes-lpddr5x-smart-with-logic-unit-in-memory-lpddr5x-pim-is-3-01x-faster-than-lpddr5x-in-ai-inference-with-8x-the-bandwidth),
[SK hynix GDDR6-AiM](https://videocardz.com/press-release/sk-hynix-announces-gddr6-aim-accelerator-in-memory-technology),
[H100](https://www.abhik.ai/concepts/gpu-computing/hbm-memory).

Read the table as a ladder, not a race. PIM is not trying to beat HBM. It is
trying to give a *commodity, JEDEC-standard* memory part — the kind soldered
to a laptop or a phone — bandwidth in the neighborhood of a datacenter GPU,
for decode, without HBM's price, power, or thermals. The pitch is that the
16 GB [LPDDR5X-PIM part is a standard 561-ball package](https://ben3d.ca/blog/processing-in-memory):
it drops into the socket the memory already uses.

Why it works at all is the memory wall itself. [Gholami's numbers](https://arxiv.org/abs/2403.14123):
over 20 years peak server FLOPS scaled **3.0× every two years**, while DRAM
bandwidth scaled **1.6×** and interconnect **1.4×**. Compute pulled away from
the pins and never came back. PIM's move is to stop crossing the pins.

## The measured wins, and the asterisks

The Samsung result is the cleanest because it is a real part, not a simulator.
Llama-3.1-8B, 320-token context, INT8 activations and INT4 weights: the
conventional build ran at 27 tokens/sec; the PIM build finished the same work
in 5.4 seconds at **81.3 tokens/sec** — 2.28× on wall-clock, **3.01× on
throughput**. The wall-clock number is lower than the throughput number
because prefill (compute-bound) doesn't benefit; only decode does.

The earlier parts point the same way. Samsung's HBM2 generation,
[Aquabolt-XL](https://ieeexplore.ieee.org/document/9749869/) — a 16-lane FP16
SIMD unit per bank, ~1.2 TFLOPS — improved microkernel GEMV by **8.9×** and
cut energy **over 60%** on a GPU host. [SK hynix's AiMX card](https://news.skhynix.com/sk-hynix-debuts-first-gddr6-aim-accelerator-card-aimx-for-generative-ai/)
claims **>10× faster** LLM processing at **one-fifth the power** of a
GPU system.

Now the asterisks, because the whole argument turns on them. Every figure
above is a vendor benchmark on a workload the vendor chose. SK hynix's 10×
carries an explicit condition — it holds *if* the control hub "is developed as
an ASIC," i.e. it is a projection of a part not yet in the form measured. And
each number is the memory-bound decode slice in isolation; none is an
end-to-end serving benchmark against a batched GPU. Treat them as the ceiling
PIM reaches on the exact operation it was built for, not as a system win.

## The operation it was built for is the only one that matters

That operation is GEMV — matrix times *vector*. In decode you multiply the
weight matrices against a single token's activation vector: one column of work,
every weight read once, arithmetic intensity ≈ 1. That is precisely the
regime the roofline says is bandwidth-bound, and precisely where PIM's
bank-local bandwidth pays.

GEMM — matrix times *matrix* — is the opposite. Prefill, training, and any
batched request stack many vectors into a matrix, so each weight read feeds
many multiplies. Arithmetic intensity climbs toward the ridge, the work
becomes compute-bound, and PIM's tiny per-bank multipliers lose to a GPU's
tensor cores by a mile. This is not a flaw to be engineered out; it is the
shape of the silicon.

So the honest architecture is a split, and the PIM vendors say so. [SK
hynix's disaggregated design](https://hc2023.hotchips.org/assets/program/conference/day1/PIM/HC35_SKhynix_DSM_YongkeeKwon_rev3.pdf)
runs the memory-bound decode phase on the AiM accelerator and the compute-bound
prefill on the GPU. PIM is a decode co-processor. It never replaces the
matrix engine; it offloads the one phase the matrix engine is bad at.

## Why fifteen years, and why maybe still not

The physics has been right since the first PIM papers. The reason it stayed on
slides is that a memory part is worthless until the software above it targets
the part, and three walls have kept the software away.

**The batching wall — the important one.** PIM's entire advantage is batch-1.
But batch-1 is the regime a datacenter engineers *away*. Serving providers
batch dozens of requests precisely to amortize the weight read across many
tokens — the same move that lifts arithmetic intensity off the floor and makes
decode compute-bound again. In a batched datacenter, the wall PIM attacks
has already been half-dismantled by scheduling, for free. So PIM is not a
datacenter play. It is a bet on the *un-batchable* token: one user, one
stream, on a device, where you cannot borrow a neighbor's request to share the
weight read and latency is the product. That market exists — local and edge
inference, the thing a phone or laptop does alone — but it is not where the
inference dollars currently are.

**The substitutes wall.** The un-batchable token is already under attack by
cheaper software. [Mixture-of-experts, 06-21](./2026-06-21-mixture-of-experts-active-parameters.md)
moves ~5% of the bytes per token, so a sparse model on ordinary memory can beat
a dense model on PIM. [Speculative decoding, 06-24](./2026-06-24-speculative-decoding-idle-compute.md)
verifies several tokens per weight read, raising intensity without new
silicon. Quantization shrinks the bytes directly. None of these need a new
memory standard, a new allocator, or a new runtime. PIM has to beat them on
tokens-per-dollar, not just on tokens-per-second.

**The standard wall.** A phone-grade PIM part is stranded without a JEDEC ISA
the compiler can target, a bank-aligned memory layout the allocator honors
(today's [block-scaled K-quants fight the tiny per-bank MAC units](https://ben3d.ca/blog/processing-in-memory)),
and a backend in llama.cpp or vLLM. This is the wall that just moved: a
[JEDEC LPDDR6-PIM standard is reported near completion as of April 2026](https://ben3d.ca/blog/processing-in-memory).
A standard is necessary. On its own it is not sufficient — the open-source
runtime integration is still pending, and a standard nobody's kernels target is
a spec, not a product.

## What the money is doing

There is a reason both memory makers are pushing now, and it is the 08-23
supercycle read from the other side. Memory has gone from
[52% of AI-chip component spending in Q1 2024 to 63% by Q4 2025](https://ben3d.ca/blog/processing-in-memory).
The value of the machine is migrating into the memory. If the margin now lives
in the DRAM, the memory makers would very much like the *compute* to live there
too — it turns a commodity part they sell on price into a differentiated part
they sell on performance. That is a strong reason for Samsung and SK hynix to
build PIM. It is not, by itself, a reason it wins. Watch what the buyers do,
not what the sellers ship.

## The deciding quantity

PIM's physics is real and measured: 8× the bank-internal bandwidth, 3.01×
tokens/sec on a shipping part, on exactly the GEMV-bound decode phase the
roofline says is starved. That is not the question. The question is whether
the *un-batchable token* is a real market or a rounding error. If a large and
growing share of inference is served batch-1 on the edge — where you can't
batch, where HBM's power budget doesn't fit, and where latency is the whole
product — PIM's 3× is a durable advantage on the axis that still has spread
([latency, 08-08](./2026-08-08-gpu-swaps-models-amd-chip.md)). If inference
stays overwhelmingly batched in datacenters, the wall PIM climbs is one the
scheduler already lowered, and PIM stays a demo with a JEDEC number.

So the number to watch is not a bandwidth figure. It is the **fraction of
inference tokens served batch-1 on-device versus batched in a datacenter**, and
whether a second-source vendor ships to the LPDDR6-PIM standard with a
merged-upstream runtime backend. Until a `llama.cpp` PIM path exists that a
normal developer can flip on, the 3.01× is a controlled result, not a product.

What would change my mind: an open-source inference runtime landing a PIM
backend that reproduces even half the vendor speedup on a stock quantized model
— that would turn "the memory can do math" from a Hot Chips slide into a line
in your build config. Watch for it. Don't budget for it yet.
