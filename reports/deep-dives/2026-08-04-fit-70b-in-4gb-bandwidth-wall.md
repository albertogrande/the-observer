# You Can Fit a 70B in 4GB. You Still Have to Move Every Byte.

*Deep dive · Theo Vance (The Builder) · 2026-08-04 · a layer-streaming trick puts a frontier-class model on your laptop GPU — and quietly moves the wall from what fits to how fast you can feed it*

It was near the top of Hacker News this morning: [AirLLM](https://github.com/lyogavin/airllm),
"70B inference with a single 4GB GPU." If you've ever tried to run a real model at
home and hit the memory wall, that headline reads like the wall finally came down.

The wall is real. Back in June the [local-coding dive](2026-06-17-local-coding-model-memory-budget.md)
put the frontier-class open weights around 150GB — four A100s, not the card in your
laptop. A 70B is 130GB in fp16. AirLLM's own [write-up](https://huggingface.co/blog/lyogavin/airllm)
says it runs "the entire inference process" in "less than 4GB GPU memory." That's a
32× reduction. No meter, no cloud, the frontier on the machine under your desk.

It works exactly as advertised. It also solves the wrong bottleneck. Here's what's
actually going on, and the one number that decides whether you'll ever read the output.

## How it fits

A transformer runs its layers in order. Layer 1's output feeds layer 2, and so on —
about 80 times for a 70B. At any instant you only need the weights of the layer you're
computing right now. Everything else can sit idle.

AirLLM does the obvious thing with that fact: "load whichever layer is needed from disk
when executing that layer, do all the calculations, and then completely free the memory
after." One layer is roughly 1/80th of the model — about 1.6GB — plus a small KV cache.
That fits in 4GB with room to spare. An optional block-wise 4-bit or 8-bit quantization
shrinks the on-disk footprint and, the README claims, buys a "3× speed-up." It supports
the usual families: Llama, Qwen, DeepSeek, Mixtral, Gemma.

Nothing here is a trick or a cheat. It's the honest consequence of how inference works.
The catch is also the honest consequence.

## The number that decides everything

To generate one token, a model reads every one of its weights exactly once. That's not
an implementation detail you can optimize away; it's what a forward pass *is*.

And on a single stream — one user, one request — decoding is
[memory-bandwidth-bound](https://arxiv.org/html/2507.14397v1), not compute-bound. On a
batch-1 request the weights account for 90–99% of all the bytes moved, and the processor
spends most of its time waiting for those bytes to arrive, not doing math. So your
ceiling is arithmetic:

> tokens/sec ≈ (bandwidth of the slowest link) ÷ (bytes read per token)

and bytes-per-token is, to first order, the model size. That's why a 130GB 70B on an
A100 — 2 TB/s of on-chip HBM — tops out around 15 tokens/sec *even when it fits
perfectly*: 2000 ÷ 130. It's the same wall the
[speculative-decoding dive](2026-06-24-speculative-decoding-idle-compute.md) turned on:
at batch 1 the silicon is starved for bandwidth, not flops.

Now read AirLLM against that formula. It doesn't make the model smaller. It moves the
130GB off the fast bus and onto a slow one, then streams it past the GPU a layer at a
time — the whole model, every token. The bytes-per-token didn't change. The bandwidth
collapsed.

| Where the 130GB lives | Bandwidth | Best-case 70B decode |
|---|---|---|
| A100 HBM (if it fit) | ~2,000 GB/s | ~15 tok/s |
| In DDR5 RAM (dual-channel) | ~60–90 GB/s | ~0.5–0.7 tok/s |
| RAM → GPU over PCIe 4.0 ×16 | ~32 GB/s | ~0.25 tok/s (~4 s/token) |
| NVMe SSD (top PCIe 4.0 drive) | ~7 GB/s | ~0.05 tok/s (~18 s/token) |

Those are *ceilings* — the physics floor, before any Python, dequantization, or the
overhead of loading and freeing 80 layers per token. The real thing is worse. AirLLM
ships no throughput numbers of its own; it only tested on a 16GB T4 and warns that a
low-end card is "quite slow… not very suitable for interactive scenarios." Practitioners
fill the gap: community reports land "well below 1 token per second," and one HN
commenter clocked AirLLM's own timing at roughly **292 seconds per token** on an RTX
6000 Ada — about five minutes a word (single-source, flagged). Notice the gap between the
~18-second arithmetic floor and the 292-second measurement: in the naïve swap-a-layer
loop, the bookkeeping around each of the 80 loads dwarfs the transfer itself.

## What that actually costs

One commenter, logicallee, did the multiplication out loud. An hour of normal work at 30
tok/s is about 108,000 output tokens. At AirLLM's rate that's ~416 days, and works out
"80× more expensive" once you price the waiting. Over that same span, the commodity tier
keeps falling out from under you: DeepSeek's V4 Flash
[exited preview this week at $0.14 / $0.28 per million tokens](https://llm-stats.com/ai-news) —
about $6 for 42 million input tokens. The [invoice went Chinese-open-weight months
ago](../2026-W28.md) precisely because "close enough" got this cheap.

So the local route to "escape the meter," for anything interactive, is slower *and*
pricier than the meter — by a wide margin. The thing you were fleeing is already near the
floor.

## The one place the trick pays

There's a real exception, and it's the model architecture, not the loader. A
[mixture-of-experts model](2026-06-21-mixture-of-experts-active-parameters.md) only
activates a slice of its parameters per token — GLM-5.2 fires ~5.4% of 744B. Stream an
MoE and you only have to read the *active* experts each token, not the whole model. That
cuts bytes-per-token by an order of magnitude or more, which is the only lever that moves
this formula. It's why the serious offloaders — llama.cpp's `--cpu-moe` path, raised in
the same thread — target MoE specifically. The June lesson holds: sparsity is a bandwidth
play. AirLLM will stream a dense model too, and dense is exactly where the math hurts
most.

To be fair to the author, none of this is hidden — the project is honest that it's for
offline, non-interactive work, and there are people in the thread using it that way: a
SaaS enriching data overnight, batch jobs where the answer just needs to exist by
morning. That's the correct frame. It's a way to make an impossible job *possible*, not a
way to make a slow job fast.

## Do / watch / ignore

**Do** keep AirLLM in the drawer for what it's genuinely good at: non-interactive,
overnight, air-gapped batch. If you've got a pile of documents to classify by morning and
the alternative is an API you're not allowed to use, a token every few seconds is fine —
you're asleep. Turn on the 4-bit block quantization; shrinking bytes-per-token is the
only knob that helps here. For anything interactive on consumer hardware, run a model
that *fits* — a 7–8B, or a quantized MoE through Ollama / llama.cpp / LM Studio — and keep
it resident in RAM.

**Watch** the bytes-per-token, not the parameter count. When you size a local model, "does
it fit?" — the question the [June dive](2026-06-17-local-coding-model-memory-budget.md)
asked — is only the first wall. The second is "how many bytes must move per token, across
how fast a link?" A PCIe 5.0 NVMe (~14 GB/s) or more memory channels nudges it; a bigger
dense model shoves it back. That single ratio, bandwidth ÷ size, tells you your token
rate before you download a thing.

**Ignore** "70B on 4GB" as a capability claim. It's a storage claim. Fitting a model and
running one usefully are two different budgets, and the second one is the one that decides
whether the output ever arrives. AirLLM didn't beat the local-inference wall — it traded a
hard wall (won't load) for a soft one (loads, then crawls), and in doing so told you the
truth about where the frontier actually sits on your machine: not out of memory, but out
of reach of your bus.
