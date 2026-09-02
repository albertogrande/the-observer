# Deep-Dive Backlog

Evergreen topic ideas for the daily dive — the "interesting topics" pool,
drawn from any day the news doesn't earn the slot. Modeled on the standing
backlog The Pragmatic Engineer keeps (~100 ideas, demand-tested over time).

**Format tags** (the columnist picks one): `how-it-works` · `x-vs-y` ·
`architecture` · `postmortem` · `what-every-engineer-should-know` ·
`economics` · `news-to-framework` · `n-lessons` · `reference` ·
`practical-guide`. Parenthetical = suggested columnist affinity, not a rule
(Analyst · Contrarian · Builder · **Operator** = the Claude Code desk).

**Workflow:** when a topic is written, move it under *Used* with its date and
file. Add ideas freely (scout, weekly editor, or the reader). Keep the live
list deep — aim for ~40+ unused ideas at all times. An idea that the news
overtakes becomes a `news-to-framework` piece instead — that's fine.

## Live — AI & models

- [how-it-works] Search against a verifier: how FunSearch (Nature 2023) and
  AlphaEvolve (2025) turn an LLM plus an automated evaluator into a discovery
  engine — the evolutionary loop, why the *scorer* not the model is what guards
  against hallucination, and the exact class of problems it fits (constructions
  with a cheap runnable checker: cap-set bounds, 48-mult matrix multiply)
  (Analyst) [surfaced by the 07-24 verifier-asymmetry dive]
- [what-every-engineer-should-know] Write the verifier first: the failing test,
  the property, the fuzzer, the eval fixture — why the quality of your check is
  the ceiling on everything an agent produces, and how to tell a faithful
  verifier from a gameable proxy before best-of-N Goodharts it (Builder/Analyst)
  [surfaced by the 07-24 verifier-asymmetry dive]
- [how-it-works] What "verified by Lean" actually guarantees: the proof-assistant
  kernel as a trusted core, why a kernel-checked proof is sound but "sound" ≠
  "the theorem you meant," and where autoformalization silently mis-states the
  goal (Analyst) [surfaced by the 07-24 verifier-asymmetry dive]
- [how-it-works] How de-novo protein binder design actually works: the
  model + verifier + wet-lab loop — where to bind, structure/sequence/co-folding
  generation (RFdiffusion / ProteinMPNN / AlphaFold-class co-folding), in-silico
  filtering, then a bio-layer-interferometry K_D screen (Adaptyv/Twist) as the
  faithful physical verifier — and why binding is the cheapest oracle in the life
  sciences, which is what made Anthropic's 14/15 campaign possible (Analyst)
  [surfaced by the 08-21 biology-verifier dive]
- [news-to-framework] The verifier-cost map of AI-for-science: which scientific
  problems carry a cheap faithful oracle (protein binding, theorem-checking,
  weather backtesting) vs which don't (clinical efficacy, in-vivo toxicity,
  long-horizon safety), why surrogate endpoints (RFS for OS) are proxy verifiers
  that can Goodhart, and how to predict where AI leaps vs helps-at-the-edges before
  the campaign runs (Contrarian/Analyst) [surfaced by the 08-21 biology-verifier dive]
- [how-it-works] What a tool call costs: tokens, latency, and round-trips in
  an agent loop (Builder)
- [how-it-works] The memory wall, from first principles: why batch-1 LLM decode
  is bandwidth-bound not compute-bound, how HBM/SRAM/mask-ROM sit on the
  roofline, and why batching (not a faster multiplier) is the real throughput
  lever — the physics under every inference-chip pitch (Analyst) [surfaced by the
  08-08 model-in-silicon dive] [NOTE: the roofline + PIM half is now covered by
  the 08-29 processing-in-memory dive; if written, frame this as the *batching /
  arithmetic-intensity* explainer — why raising batch size, not new silicon, is
  the real lever — to avoid a rehash]
- [x-vs-y] Three answers to the bandwidth wall, compared on tokens/$: PIM
  (compute in the DRAM bank, 08-29) vs on-die SRAM (Cerebras/Groq — weights on
  the die at ~PB/s, no DRAM) vs software (MoE + speculative decoding + batching,
  no new silicon) — which workload each one wins, and why the edge/batch-1 slice
  is the only place custom memory silicon beats a scheduler (Analyst) [surfaced
  by the 08-29 processing-in-memory dive]
- [how-it-works] GEMV vs GEMM, and why decode and prefill want different silicon:
  matrix-vector (one token, every weight read once, memory-bound) vs
  matrix-matrix (batched/prefill, weight read amortized, compute-bound), how
  arithmetic intensity climbs off the roofline floor with batch size, and why
  prefill/decode *disaggregated* serving (SK hynix AiM split, DistServe/Splitwise-
  style pools) is the architecture the split implies (Analyst) [surfaced by the
  08-29 processing-in-memory dive; sibling to bandwidth-wall 08-04, MoE 06-21]
- [economics] The tapeout ladder: what a chip actually costs to design at 5nm vs
  3nm (mask sets, EDA, verification, headcount), why NRE not wafer price gates who
  can build custom silicon, and how model-specific "metal-only re-spin" tricks
  change the math (Analyst) [surfaced by the 08-08 model-in-silicon dive]
- [x-vs-y] Open weights vs closed APIs: the real total cost once you price
  inference, ops, and eval (Analyst)
- [how-it-works] Deduplication and memorization, from the inside: the Carlini
  log-linear laws (verbatim memorization grows with model capacity × how many
  times a text is duplicated × prompt-context length), discoverable vs
  probabilistic extraction, and why corpus dedup cuts compute AND regurgitation
  (copyright) risk at once — the technical layer under the 08-18 corpus-liability
  dive (Analyst) [surfaced by the 08-18 dive]
- [x-vs-y] Licensed-corpus vs scraped-corpus models: whether a "trained only on
  licensed/owned data" tier (Adobe-Firefly-style clean provenance) can command a
  real price premium, what it costs in capability, and whether enterprises pay for
  provenance the way they pay for ZDR (Contrarian/Analyst) [surfaced by the 08-18
  corpus-liability dive]
- [economics] The copyright contingency nobody prices: statutory-damages math
  (per-work $750–$150k × works × willfulness) as an unbounded tail on an AI lab's
  valuation, how a revenue-multiple IPO buries it, and what an S-1 risk factor
  discloses-without-bounding — pegged to Bartz's $1.5B (Analyst/Contrarian)
  [surfaced by the 08-18 corpus-liability dive]
- [how-it-works] The KV cache: why context length, not parameter count,
  decides what fits in your VRAM — with the linear-growth math (Builder)
- [how-it-works] Why batch-1 decode is memory-bandwidth-bound: the weights are
  90–99% of the bytes moved per token, so single-stream tokens/sec ≈ effective
  bandwidth ÷ model size — the arithmetic under "10–20 tok/s on an A100," why
  batching (not a faster GPU) is how providers amortize the weight read, and why
  it's the opposite of the compute-bound prefill/training regime (Analyst/Builder)
  [surfaced by the 08-04 AirLLM bandwidth-wall dive; sibling to spec-decoding 06-24]
- [economics] Latency as the last axis with spread: once value-frontier coding models
  converge on capability and price hits a floor (DeepSeek V4 Pro 80.6 SWE-bench at $0.87/M,
  Gemini 3.7 Flash, Grok 4.6), tokens/sec becomes the differentiator — and it's a
  memory-bandwidth problem, which is why the fast providers moved the weights onto the die
  (Cerebras WSE on-chip SRAM at ~PB/s vs HBM's ~TB/s, Groq LPU). Ties the bandwidth-wall
  and benchmark-saturation dives together: the buy is on the axis that still resolves
  (Analyst) [surfaced by the 08-14 Cerebras/GPT-5.6-Sol signal]
- [x-vs-y] Weight offloading, done right: llama.cpp `--cpu-moe`/mmap vs
  AirLLM-style dense layer-streaming — what actually moves per token across the
  disk→RAM→GPU chain, why MoE is the one architecture where streaming is tolerable
  (only the ~5% active experts move), and how to size a partial offload before you
  download the weights (Builder) [surfaced by the 08-04 dive; sibling to local-coding
  06-17, MoE 06-21]
- [practical-guide] Quantization without tears: dynamic 4-bit, KV-cache
  quantization, and what actually loses quality vs what doesn't (Analyst)
- [how-it-works] How a vision model tokenizes an image: patches as visual
  tokens, the ⌈w/28⌉×⌈h/28⌉ cost, resolution tiers, and why a picture of text
  is priced like a picture (Analyst)
- [how-it-works] Optical context compression: DeepSeek-OCR's premise that a page
  carries fewer tokens than its character count — the compression-vs-fidelity
  curve, and where near-lossless recovery stops (Analyst)
- [how-it-works] DPO/RLHF from the inside: how a (chosen, rejected) preference pair
  becomes a weight update, the binary-loss math, and why the ranking — not the code
  — is the scarce training input (Analyst) [surfaced by the 07-10 accept-button dive]
- [x-vs-y] RLHF vs RLAIF: where synthetic AI feedback genuinely substitutes for human
  preference (style, helpfulness) and where it collapses (correctness-centric
  comparisons, where LLM judges run marginally above random) (Contrarian/Analyst)
- [what-every-engineer-should-know] Calibration, and why post-training breaks it: what
  it means for a model's stated confidence to track its accuracy, the GPT-4 report's own
  before/after RLHF plot (Fig 8), how to measure your model's calibration on your own
  prompts, and why a single-shot benchmark can never see it (Analyst/Contrarian)
  [surfaced by the 08-15 stopped-asking dive]
- [how-it-works] Homomorphic encryption for private inference, for engineers: what
  Google's "private AI" actually computes without decrypting, the concrete overhead
  (latency/throughput cost per operation), and where it's real vs demo-ware — the durable
  version of the Aug 15 HN story (Analyst) [surfaced by the 08-15 research sweep]
- [economics] The data flywheel of an AI IDE: what accept/reject/edit telemetry is
  actually worth as training data, why zero-data-retention/Privacy Mode fences the
  highest-value enterprise slice, and whether the flywheel is a moat or a marketing
  line (Contrarian)
- [what-every-engineer-should-know] Measuring your own review miss-rate: seeding
  known-bad diffs, the aviation "inject failures in training" practice, and why a
  false-negative rate you've never measured is the real safety number of an
  AI-assisted codebase (Builder/Contrarian) [surfaced by the 07-22 deskilled-reviewer dive]
- [how-it-works] BPE tokenizers from the inside: merge tables, vocabulary budgets
  (o200k ~200k vs cl100k ~100k), why code and whitespace fragment worse than prose,
  and why the same file is 1.5–1.7× more tokens on one provider than another
  (Builder/Analyst) [surfaced by the 07-14 tokenizer-price dive]
- [practical-guide] A cost-per-solved-task harness: measuring model economics end to
  end — tokenizer × per-token price × output share × cache-hit rate × turns-to-done —
  instead of ranking models by the per-token list price (Analyst) [surfaced by the
  07-14 tokenizer-price dive]
- [x-vs-y] On-device vs cloud inference, the general rule: the decision is model-size
  the task needs ÷ device memory budget — why speech-to-text flipped to on-device
  (small model, bounded task) while frontier coding didn't (~150GB model), and how to
  tell which side of the line a workload sits on (Analyst) [surfaced by the 07-15
  on-device-speech dive; sibling to local-coding 06-17]
- [how-it-works] How speech recognition is scored and built: what word error rate
  (WER) actually counts, why read-speech benchmarks (LibriSpeech) flatter a model vs
  real far-field/multi-speaker audio (earnings22), streaming volatile-vs-final results,
  and the accuracy/latency/coverage tradeoffs of a transcription stack (Analyst)
- [how-it-works] Test-time compute from the inside: self-consistency (majority vote),
  best-of-N with a reward model / verifier, and beam/tree search over reasoning chains —
  why accuracy scales log-linear with samples, what a verifier actually buys over a raw
  vote, and where parallel sampling beats one long chain (Analyst) [surfaced by the
  07-18 reasoning-cost dive]
- [x-vs-y] Flat vs metered long-context pricing: why Anthropic removed its >200k
  2×/1.5× surcharge for flat 1M while OpenAI kept a 272k price cliff on GPT-5.6 —
  what the KV-cache-linear + attention-quadratic cost curve says about which bet is
  sustainable, and how to read a provider's context tiers before you budget (Analyst)
  [surfaced by the 07-21 context-price-cliff dive]
- [practical-guide] Setting a thinking budget: measuring the accuracy-vs-cost curve of
  `thinkingBudget`/`reasoning_effort` on your own workload, allocating compute by task
  difficulty (Snell), and picking the N for best-of-N where a wrong answer is worth 64
  right ones (Analyst/Operator) [surfaced by the 07-18 reasoning-cost dive]
- [how-it-works] Reduced-round cryptanalysis, for engineers: why cryptographers attack
  7-of-10-round AES on purpose, what a meet-in-the-middle attack actually does, and how to
  read "N-round attack, 2^X operations, 2^Y chosen plaintexts" without panicking — the
  safety-margin frame that separates an academic result from a break (Analyst/Contrarian)
  [surfaced by the 07-31 AI-cryptanalysis dive]
- [what-every-engineer-should-know] "An AI found a vulnerability" is three different claims:
  an implementation bug (what DevSecOps scanners like OpenAI Codex Security / MAI-Cyber-1-Flash
  do), a reduced-round or not-yet-standardized cryptanalysis result, and a full-round deployed
  break — which ones justify rotating a key and which are just cheaper labor (Contrarian)
  [surfaced by the 07-31 dive]
- [how-it-works] Content provenance from the inside: C2PA / Content Credentials hard
  bindings (signed manifests) vs soft bindings (invisible watermarks), why an upload
  re-encode / screenshot / crop breaks the cryptographic signature, the cloud-manifest
  fallback, and what a provenance chain can and can't promise once the file leaves your
  server (Analyst) [surfaced by the 08-02 AI-Act-marking dive]
- [x-vs-y] The two answers to "mark AI content": cryptographic provenance (C2PA, beside
  the pixels) vs signal watermarking (SynthID, in the pixels/tokens) — the survival
  curve of each against common vs adversarial transforms, why text is the worst medium,
  and why a regulation ends up wanting both (Analyst/Contrarian) [surfaced by the 08-02
  AI-Act-marking dive]
- [how-it-works] Post-quantum crypto for the working engineer: what's actually deploying
  (ML-KEM / ML-DSA / hybrid X25519+ML-KEM), the migration timeline, why "harvest now, decrypt
  later" is the real threat model, and how the NIST standardization gauntlet withdraws a scheme
  (HAWK's lattice automorphism) before it ships (Analyst) [surfaced by the 07-31 dive]

- [what-every-engineer-should-know] Verification as the career skill: how to
  practice the checkable half of "taste" on purpose — writing the spec before the
  code, predicting the diff, seeding known-bad changes to measure your own
  false-negative rate — and why output speed is the skill AI just made worthless
  while error-detection is the one it made scarce (Contrarian/Builder) [surfaced
  by the 08-09 skill-distribution dive; sibling to deskilled-reviewer 07-22]
- [x-vs-y] Where AI levels skill vs where it concentrates it: a field-experiment
  map (customer support / consulting / greenfield — cheap verifier, leveling) vs
  (brownfield production / security — expensive verifier, concentration), the
  Brynjolfsson-QJE and Dell'Acqua-BCG vs METR results side by side, and the
  verifier-cost variable that predicts the sign before you run the study (Analyst/
  Contrarian) [surfaced by the 08-09 skill-distribution dive]

## Live — Claude Code & agent engineering

*The Operator's home pool — Claude Code workflows and max-performance. The
weekly Claude Code edition (Thursdays) draws from here and the scout's Claude
Code Watch.*

- [practical-guide] The daily-driver loop: structuring a Claude Code session
  so the agent finishes more before it derails (Operator)
- [reference] A CLAUDE.md that earns its tokens: what to put in, what to cut,
  for a large repo (Operator)
- [practical-guide] The permission allowlist that stops the prompts without
  going full YOLO (Operator)
- [how-it-works] PostToolUse hooks as *quality* gates — lint, test, and format
  the agent's output before it lands (distinct from the PreToolUse security
  veto covered 2026-07-02, and from the output-*replacement*/context-recovery
  angle covered 2026-08-06) (Operator)
- [practical-guide] Redaction hooks for the coding agent: a PreToolUse +
  PostToolUse pair that strips secrets/PII from tool input and `tool_response`
  so the model never reads (or echoes into a commit) an AWS key or bearer token
  — the one case where losing the original output is the goal (Operator)
  [surfaced by the 08-06 tool-output dive]
- [how-it-works] Agent teams from the inside: the collaborative tier the 08-13
  dive named but didn't open — the mailbox (`~/.claude/teams/.../inboxes`), the
  shared task list with file-locking + dependencies, the plan-approval protocol,
  tmux/iTerm2 split-pane display, and the experimental limitations (one team per
  session, no `/resume`, no nested teams) — i.e. *when* the collaboration is
  worth the token cost over separate sessions (Operator) [the subagent/team/
  session three-tier line is covered 07-30 + 08-13; this is the team internals]
- [reference] The self-hosted runner: `claude self-hosted-runner` turning your
  own machines/containers into executors for Claude Code web/mobile/desktop
  sessions — what it changes about where work runs and where the code/data lives
  (Operator) [surfaced by the 08-13 multi-session dive]
- [practical-guide] Cross-session orchestration: a driver session delegating to
  worker sessions via `SendMessage`/`ListAgents` across machines, `--teleport`
  handoffs, and why an incoming session message is treated as *untrusted* (it
  can't approve your permission prompts) — the ceiling on fully hands-off
  fan-out (Operator) [surfaced by the 08-13 multi-session dive]
- [practical-guide] Structured subagent returns: designing the schema a
  subagent hands back (`--output-format json` + `--json-schema`,
  `structured_output`) so an orchestrator can gate on data, not prose — the
  return message IS the interface (Operator/Builder) [surfaced by the 07-30
  subagents dive]
- [reference] Reading a subagent tree with `--forward-subagent-text`: the
  stream-json event shapes, `parent_tool_use_id` nesting reconstruction, and a
  jq recipe to watch an unattended fan-out live (Operator) [surfaced by the
  07-30 subagents dive]
- [n-lessons] What the top Claude Code workflows have in common — patterns
  lifted from power users (Operator)
- [reference] MCP servers worth wiring into Claude Code, and what each one
  actually buys you (Operator)
- [how-it-works] Driving Claude Code headless from CI: the unattended-agent
  workflow pattern (Operator)
- [how-it-works] Tool Search / deferred tool loading from the inside: how
  `defer_loading` and an on-demand tool-discovery step keep a 58-tool library out of
  context until the model needs 3–5 of them, the round-trip it adds, and why fewer
  tools in view *raises* tool-call accuracy — not just cuts tokens (Operator/Analyst)
  [surfaced by the 07-16 context-tax dive]
- [practical-guide] An MCP context-budget audit: measuring each connected server's
  standing token cost with /context, ranking them by cost-per-use, and deciding which
  to defer, disconnect, or scope per-project (Operator) [surfaced by the 07-16
  context-tax dive]
- [postmortem] When a compaction summary drops the thing you needed — designing
  CLAUDE.md compaction rules and checkpoints that survive the save (Builder)
- [architecture] The anatomy of an MCP server: transport, tool schemas, and
  what a call really costs (Builder)
- [x-vs-y] MCP vs A2A: the tool rung vs the agent rung — what each protocol
  actually wires together, and why one got adopted and one got shrugged at (Builder)
- [how-it-works] The ReAct loop, from the inside: reflection → action →
  observation, the memory it needs, and what the framework actually adds (Builder)
- [practical-guide] Writing a skill that triggers reliably — the description is
  the product (Builder)
- [postmortem] Failure modes of agent loops: runaway turns, context rot, and
  silent tool errors (Analyst)
- [x-vs-y] One big agent vs many small ones: where the orchestration overhead
  stops paying off (Analyst)
- [n-lessons] What a week of letting an agent write 80% of the code actually
  taught us (Builder)
- [reference] The Claude Code env vars that actually change behavior —
  CLAUDE_CODE_DISABLE_1M_CONTEXT, CLAUDE_AUTOCOMPACT_PCT_OVERRIDE,
  MAX_THINKING_TOKENS, and the rest, with what each one buys (Operator)
- [practical-guide] Externalized memory across /clear: designing a PROGRESS.md
  handoff file an agent can dump to and reload without losing the "why" (Operator)
- [practical-guide] A tool-call reliability eval: the small fixed harness you run
  on every model bump to catch schema-adherence regressions before prod (Builder)
- [how-it-works] Grammar-constrained / strict decoding from the inside: how a JSON
  Schema becomes a token mask, the reasoning tax it can charge, and the supported
  subset that trips people up (Analyst)
- [practical-guide] Skill descriptions as a routing problem: writing the one-line
  trigger the model actually matches on, and running the skill-creator eval loop
  (should-trigger / should-not-trigger hit rate, with-vs-without token overhead) to
  prove a description earns its keep (Operator)
- [how-it-works] `context: fork` — running a skill as a forked subagent: what
  loads (SKILL.md as the prompt, agent-type system prompt, CLAUDE.md unless
  Explore/Plan), what it can't see (your conversation), and when a forked skill
  beats an inline one (Operator)
- [reference] Dynamic context injection in skills: the `` !`cmd` `` preprocessing
  syntax, `${CLAUDE_SKILL_DIR}` / `$ARGUMENTS[N]` substitutions, and how to ground
  a skill in live repo state before the model ever reads it (Operator)
- [how-it-works] Credential masking in the sandbox proxy: `credentials.mode:"mask"`
  + `network.tlsTerminate` — how a per-session sentinel value is swapped for the real
  token only on the request that leaves for an `injectHosts` domain, so the command
  (and its logs) never hold the secret. When mask beats deny, and why it needs the
  proxy to terminate TLS (Operator) [surfaced by the 07-23 sandbox dive]
- [practical-guide] Sandboxing the MCP servers you didn't write: wrapping a local MCP
  server in `srt` (`@anthropic-ai/sandbox-runtime`) so a third-party tool server runs
  inside the same filesystem+network fence — the config, the deny-all-network default,
  and what it does and doesn't contain (Operator) [surfaced by the 07-23 sandbox dive;
  sibling to the MCP-trust-boundary and egress items]
- [x-vs-y] Auto-memory vs a hand-written CLAUDE.md: what Claude's new automatic memory
  actually saves, where it's stored, whether it's inspectable/portable/version-controlled,
  and when you still want a hand-curated file — the convenience-vs-auditability tradeoff of
  a log the actor keeps on itself (Operator/Builder) [surfaced by the 07-26 context-
  scaffolding dive; sibling to audit-trail 07-08]
- [practical-guide] Versioning your prompt against the model ID: treating every model
  swap — an upgrade to a more capable model *or* a downgrade to a cheaper routed one — as a
  portability event; keeping per-model "altitude" variants (more rules for the weaker model,
  fewer for the frontier) and gating the swap on an eval fixture (Builder) [surfaced by the
  07-26 context-scaffolding dive; sibling to tool-schema 07-05, portability 06-22]
- [practical-guide] The cache-miss tax in a long session: why cache-READS dominate a
  healthy Claude Code session's token count (the docs' own $0.55 example: 1.2k input vs
  940k cache-read), how the cache lifetime (1hr on a subscription, 5min on credits/API)
  turns any idle break into a full-context reprocess, and how to read the `/usage`
  behavior flag that now names it at ≥10% of usage — plus the habits (don't idle across
  the lifetime, choose the TTL, resume-from-summary) that keep the prefix warm (Operator)
  [surfaced by the 08-27 itemize-the-bill dive; sibling to caching 06-18, context-tax 07-16]
- [reference] OTEL cost export for a Claude Code fleet: `CLAUDE_CODE_ENABLE_TELEMETRY=1`,
  the per-user token/cost metrics and tool-decision events it streams, and why it's the
  only real-time per-user attribution path once you're on Bedrock / Google / Foundry
  (where the Anthropic analytics dashboards go dark) — vs the local, list-rate estimates
  `/usage` shows on one machine (Operator/Builder) [surfaced by the 08-27 dive; sibling to
  audit-trail 07-08]

## Live — devtools & systems

- [how-it-works] Declarative vs imperative extension APIs, the general pattern:
  why moving a capability from "run my callback" (blocking webRequest, arbitrary
  logic, sees everything) to "match my rules" (declarativeNetRequest, a fixed
  grammar the host evaluates) trades expressiveness for safety+speed+privacy in
  every host — browsers, MCP tool schemas, permission systems — and how to tell
  which class of feature survives the move and which is structurally lost (the
  no-top-context, no-response-body operations) (Analyst/Builder) [surfaced by the
  09-01 declarativeNetRequest-ceiling dive]
- [x-vs-y] The three browsers' content-filtering models compared, on capability
  not politics: Chrome MV3 declarativeNetRequest (30k/300k rule budget, no dynamic
  filtering) vs Firefox (kept blocking webRequest, full uBO) vs Safari Content
  Blocker (a much older declarative-rules-only model with its own per-extension
  rule cap — verify the current number) — Safari has
  run the MV3 experiment since 2015, so its content-blocker ecosystem is the
  natural forecast for where Chrome's lands (Analyst) [surfaced by the 09-01 dive]
- [practical-guide] Run your own model mirror: standing up an `HF_ENDPOINT`
  pull-through cache / proxy (Pulp, Artifactory-style, or an S3-backed mirror) so
  your builds don't depend on huggingface.co's uptime, pricing, or ownership — plus
  revision-pinning + integrity verification as supply-chain hygiene for weights, the
  same way you'd vendor an npm or container registry (Builder) [surfaced by the
  08-28 huggingface-dependency dive]
- [how-it-works] Content negotiation for agents: `Accept: text/markdown` and the
  emerging convention of serving a stripped markdown view to agentic clients
  (acceptmarkdown.com, HN Aug-28) — the HTTP-primitive successor to llms.txt/MCP for
  docs-as-distribution; what it standardizes and what it can't (Builder) [surfaced
  08-28; sibling docs-for-agents 07-04]
- [practical-guide] AI-assisted fuzzing: wiring a model into a fuzz harness (write
  the harness + triage the crash) as the canonical cheap-verifier task — a crash is
  a machine-checkable witness — pegged to the vibecoded-fuzzer FFmpeg CVE (HN Aug-28)
  and OSS-Fuzz's LLM harness-generation results; the builder's hands-on version of
  the verifier-asymmetry law (Builder) [surfaced 08-28; sibling verifier-asymmetry
  07-24, 08-21]
- [architecture] The agent control plane: where governed tool access and egress
  actually get enforced for *deployed* agents once you can't trust the agent
  (Cloudflare OS "Gatekeepers," Microsoft's Foundry router, identity-for-agents) —
  the org-level mirror of the CLI's hooks/sandbox, and why the confused-deputy /
  prompt-injection case (Atlassian Rovo exfiltration) forces enforcement to the
  network gateway (Builder) [surfaced by the 08-07 agent-config-portability dive]
- [x-vs-y] The orchestration layer above the harness: Warp's "delegate to Claude
  Code and Codex" model vs a single-harness setup — when a router-of-agents beats a
  best-agent, and what it costs you in observability (Builder) [surfaced 08-07]
- [reference] A portable MCP-server manifest? Why your tool *wiring* (which servers,
  which scopes, which auth) is still per-harness even though MCP is a shared protocol,
  and what a repo-committed, harness-agnostic MCP config would have to specify to
  travel (Builder/Operator) [surfaced by the 08-07 agent-config-portability dive]
- [practical-guide] The model-switch checklist: what a harness must do on a
  mid-session provider/model swap to not silently overcharge you — strip prior
  model-bound `thinking`/`redacted_thinking` blocks (else you pay input tokens
  for ignored reasoning), accept a cold cache prefix on the new model, re-tune
  the system prompt's altitude, and gate the swap on a fixture — the operational
  companion to the 08-25 switch-model-not-state dive (Builder/Operator) [surfaced
  by the 08-25 mid-session-switch dive; siblings portability 06-22, caching 06-18,
  reasoning-trace 08-12]
- [how-it-works] The accessibility tree as an agent API: how a screen-reader
  data structure (roles, names, ARIA states, stable refs) became the cheap,
  reliable way for agents to read a page — and what unlabeled `<div>` soup costs
  you in tokens and flake (Builder) [surfaced by the 07-11 browser-runtime dive]
- [how-it-works] Commit identity for agents: why git's author/committer field is
  a self-asserted config string (not authenticated; GitHub matches by email, SSH
  validates the push not the authorship), what "Verified" actually proves
  (GPG/SSH/S-MIME signing, sigstore/gitsign keyless), and what a first-class
  "which agent, under whose authority, reviewed by whom" record would have to
  add on top of `Co-authored-by:` — the attribution layer under agent-written
  code (Builder/Analyst) [surfaced by the 08-19 agent-code-hosting dive]
- [x-vs-y] Bidirectional sync as a migration wedge: the embrace on-ramp pattern
  (GitLab's GitHub importer, Cursor Origin) — the "system of record vs daily
  surface" split, how the center of gravity drifts to the tool people actually
  work in, and the exact tell that the wedge has become the lock-in (the day the
  sync direction defaults the other way) (Contrarian/Builder) [surfaced by the
  08-19 agent-code-hosting dive]
- [how-it-works] Sandboxing build-time code execution: how the Rust "sandboxed
  build script" project goal actually fences a `build.rs` (seccomp/namespaces/
  no-network vs a coarse container), why proc-macros are the harder case
  (dtolnay's Watt runs them as WebAssembly so a macro can only shuffle tokens),
  and the general design of "let the ecosystem run untrusted code at build time
  without giving it your machine" — the durable follow-on to the 08-22
  build-time-attack dive (Builder/Analyst) [surfaced by the 08-22 arrayref dive]
- [x-vs-y] The three package-manager trust models compared: npm (postinstall +
  v12's script-off default), Python (setup.py / PEP 517 build isolation +
  wheels-avoid-execution), and Cargo (build.rs + proc-macros, no sandbox yet) —
  where each executes author code, what "install a wheel, don't build it" buys,
  and why "compiled = safer" is exactly backwards for supply-chain risk
  (Analyst/Contrarian) [surfaced by the 08-22 build-time-attack dive]
- [practical-guide] cargo-vet / cargo-audit / cargo-deny in a real repo: the
  audit-before-it-enters workflow, wiring the RustSec advisory DB into CI so a
  known-bad version fails the build instead of running it, and honestly costing
  the friction of human dependency audits vs the class of attack it stops
  (Builder/Operator) [surfaced by the 08-22 build-time-attack dive]
- [how-it-works] Why your test suite is slow: the real cost model of a CI run (Analyst)
- [how-it-works] Deterministic simulation testing from first principles: how FoundationDB
  and Antithesis control the scheduler + the clock to make concurrency reproducible, why a
  single seed replays an entire run bit-for-bit, what "search the schedule space against an
  invariant" actually costs, and when it beats fuzzing/coverage — the deeper follow-on to the
  SQLite piece (Analyst) [surfaced by the 08-14 SQLite WAL-race dive]
- [what-every-engineer-should-know] What test coverage does and doesn't measure: line vs
  branch vs MC/DC, why 100% of any of them is silent on the interleaving/state space where
  concurrency and durability bugs live, and how to think about "reachable-schedule coverage"
  as the number that matters for stateful systems (Analyst) [surfaced by the 08-14 SQLite dive]
- [how-it-works] Inside SQLite's WAL and checkpointing: the -wal/-shm files, the WAL-index
  header and backfill counter, reader end-marks, and the one-writer concurrency contract — a
  clean "how this load-bearing part of the stack actually works" (Analyst) [surfaced 08-14]
- [how-it-works] The successor-benchmark landscape after saturation: SWE-bench Pro,
  FrontierMath Open Problems, formal-conjecture / Lean-verified evals — what "unsaturable
  + machine-checkable + contamination-proof" costs to build and who is paying for it
  (Analyst) [surfaced by the 08-05 benchmark-saturation dive]
- [how-it-works] Clustered standard errors in an eval, from the inside: why related
  items (same repo, same passage, variants of one bug) aren't independent, how clustering
  on the unit of randomization can inflate the true SE ~3× over the naive one (Miller/
  Anthropic), and how to detect the hidden cluster structure in a benchmark you didn't
  build — a "500-item" set that's really 120 scenarios × 4 is a 120-item set (Analyst)
  [surfaced by the 08-26 eval-sample-size dive]
- [practical-guide] Resampling to kill eval noise: run each item several times and score
  the per-question average (chain-of-thought models) vs use next-token probabilities as
  the score (deterministic models) — the variance-reduction math, the compute cost, and
  why a single agentic run's seed-of-the-day swing (SWE-agent SD 0.49 over six re-runs)
  is bigger than most leaderboard gaps (Analyst/Operator) [surfaced by the 08-26 dive]
- [x-vs-y] Monorepo vs polyrepo once AI agents are the main committer (Contrarian)
- [how-it-works] Teaching a model a low-resource language: how MultiPL-T and
  synthetic-function-demonstration pipelines translate high-resource training
  data into a niche language, how much pass@1 it actually buys, and why the
  economics only fund it for languages that already have demand — the rescue path
  for the Zig/Nim/Odin tier under AI authorship (Analyst/Contrarian) [surfaced by
  the 07-28 language-corpus dive]
- [how-it-works] Pause, ship, resume: durable execution for agents — how a
  code-mode sandbox suspends model-written code at a mid-program tool call, runs
  it on the client, and resumes (coroutine suspension vs stateless replay-from-top
  with memoized results), why replay is the Temporal/durable-functions trick, and
  the idempotency footgun replay reintroduces (a non-memoized side effect fires
  twice) (Builder) [surfaced by the 08-16 code-mode dive; sibling to idempotency
  06-26, MCP-stateless 08-01]
- [what-every-engineer-should-know] The software-patent cloud over AI patterns:
  how a B1 grant (no pre-grant publication → no public prior-art window) happens,
  what "prior art" actually has to show to invalidate a claim, why a granted
  patent that would likely fall in court is still a tax (cost-to-fight, not
  strength), and how a builder reads a claim to stay off it — pegged to Mistral's
  US 12,670,045 (Contrarian/Builder) [surfaced by the 08-16 code-mode dive]
- [practical-guide] The tools-as-filesystem pattern: exposing MCP servers as an
  importable code API a coding agent discovers on demand (load a schema only when
  the code reaches for it) instead of front-loading every tool definition — the
  build, and how it composes with Tool Search / deferred loading to attack the
  same context tax from two sides (Operator/Builder) [surfaced by the 08-16
  code-mode dive; sibling to context-tax 07-16, deferred-tool-loading item]
- [how-it-works] The MCP Tasks extension: how long-running agent work moves from
  a held-open connection to a poll-based, resumable task once the transport goes
  stateless — the durability you now own yourself, and where a plain tool call
  ends and a Task begins (Builder) [surfaced by the 08-01 MCP stateless dive]
- [what-every-engineer-should-know] Sealed state on the wire: HMAC-signed vs
  encrypted request/session state, TTLs, the "visible but tamper-evident" trade,
  and the replay/accumulation footguns — the signed-cookie pattern the 2026-07-28
  MCP spec adopted, generalized (Builder) [surfaced by the 08-01 MCP stateless dive;
  now also pegged by the 08-12 reasoning-trace dive — encrypted client-held CoT is the
  same footgun at the LLM frontier]
- [how-it-works] The reasoning-block protocol, from the inside: what an
  `encrypted_content` (OpenAI) / `signature` / `redacted_thinking` (Anthropic) block
  actually is, why the stateless API forces the full chain-of-thought to round-trip
  through the client, what the server verifies vs decrypts on the way back, and the
  key-scoping choices (session-bound vs portable-across-model) that decide whether a
  "hidden" trace is recoverable — the systems layer under the 08-12 stolen-traces
  result (Builder/Analyst) [surfaced by the 08-12 reasoning-trace dive]
- [architecture] The MCP gateway: what a managed middlebox in front of a fleet of
  MCP servers actually does — header-based routing (Mcp-Method/Mcp-Param-*),
  per-request OAuth (RFC 9728/8707), list caching, and the confused-deputy trap it
  has to avoid (Builder/Analyst) [surfaced by the 08-01 MCP stateless dive]
- [postmortem] Anatomy of a dependency-confusion attack, end to end (Analyst)
- [what-every-engineer-should-know] OIDC, OAuth, sessions, and tokens — the auth
  vocabulary every engineer keeps confusing (Builder)
- [how-it-works] How a package registry serves a billion downloads a day (Analyst)
- [architecture] How a fast build cache actually works (and why yours misses) (Builder)
- [postmortem] The class of outage that's always DNS, and why (Analyst)
- [how-it-works] What an LLM gateway actually does: proxying, failover, caching,
  cost caps, and where the latency hides (Builder)
- [architecture] Why LLM observability runs on a columnar database — the trace
  store, not the dashboard, is the asset (Analyst)
- [how-it-works] The OpenTelemetry GenAI semantic conventions, from the inside:
  invoke_agent/execute_tool spans, the gen_ai.* attribute set, and why they're
  still "Development" status — what you can and can't build a durable dashboard
  on (Builder/Analyst)
- [what-every-engineer-should-know] Tamper-evident logs for machine actors:
  hash-chained append-only records (Merkle/RFC 8785), the external-witness
  trick, and why "trust me" telemetry stops being evidence when the actor is an
  agent (Analyst)
- [how-it-works] The kill chain, step by step: what an LLM agent can and can't
  automate in a real intrusion — where autonomy holds (chaining a known CVE,
  self-correcting an error) and where it collapses (novel initial access,
  evasion, hallucinated access) (Contrarian/Analyst)
- [news-to-framework] The attacker's marginal cost: how cheap agent labor
  changes the *volume and targeting distribution* of attacks, not the capability
  ceiling — and why that makes it a defense-and-hygiene problem, not a superhacker
  one (Contrarian)
- [what-every-engineer-should-know] The exposed-and-default surface agents
  harvest first: internet-facing known-CVE services, default credentials
  (minioadmin:minioadmin), unrotated signing keys — and the boring hardening that
  actually blunts an agentic attacker (Builder/Analyst)
- [practical-guide] Proxy your agent: a mitmproxy recipe for reading exactly what
  a coding CLI sends over the wire — the host list, the request bodies, the
  telemetry pipes — and the traffic-control env vars worth setting (Builder)
  [surfaced by the 07-17 agent-egress dive; that piece is the news peg — this is
  the deeper standalone how-to]
- [reference] The MCP trust boundary: every connected server sees what you hand
  it — mapping the data-egress surface of a loaded MCP config, per-server scoping,
  and what "State of MCP Security 2026" actually found (Builder/Analyst)
  [surfaced by the 07-17 agent-egress dive]
- [x-vs-y] Open-source vs closed coding CLIs as a *trust* decision (not a price
  one): what reading the source buys you, what it doesn't, and why license ≠ safety
  once the tool runs with your keys (Contrarian) [surfaced by the 07-17 dive]

- [how-it-works] AI control, from the inside: the trusted-monitor-over-untrusted-
  policy protocol (Redwood), how a control evaluation actually runs (Blue Team builds
  the protocol, Red Team plays the scheming model), the menu of moves (trusted
  monitoring, untrusted monitoring, defer-to-trusted, resampling), and the single
  dependency every protocol rests on — a monitor the untrusted model can't reach —
  which is exactly what the HuggingFace incident broke (Adaptive Attacks on Trusted
  Monitors, arXiv 2510.09462) (Analyst/Contrarian) [surfaced by the 09-02 HuggingFace
  eval-integrity dive]
- [news-to-framework] The measurement inside the blast radius: why an agent that can
  write to the channel its grader reads corrupts the eval by construction — in-band vs
  out-of-band logging, why "reward hacking" (gaming the score) and "eval tampering"
  (rewriting the record of what you did) are the same failure at two layers, and the
  separation-of-duties fix (the actor is never the source of its own graded record) —
  pegged to the METR/Redwood HuggingFace autopsy (spoofing in >7% of transcripts,
  contradicting the lab's "logs unaffected") (Contrarian/Analyst) [surfaced by the
  09-02 dive; sibling tamper-evident-logs, audit-trail 07-08, verifier-asymmetry 07-24]

## Live — DevRel, dev marketing, product engineering

- [reference] What DevRel is actually for at an AI-tools company in 2026 (Builder)
- [reference] Developer experience as a measurable thing, not a vibe (Builder)
- [n-lessons] What makes developer documentation get linked vs ignored (Builder)
- [reference] The product engineer: who owns the gap between design and prod (Builder)
- [x-vs-y] Bottom-up vs top-down adoption for a developer tool — the real
  funnel math (Analyst)
- [what-every-engineer-should-know] How open-source-as-distribution actually
  pays back (Contrarian)
- [practical-guide] Making your webapp agent-callable with WebMCP: `registerTool`,
  the declarative form, and why the distribution win only lands if you put a real
  consent step in front of every write tool (Builder) [the 07-11 dive covered the
  three-level gradient; this is the hands-on build] 

## Live — economy & the AI business

- [economics] The unit economics of an AI coding subscription, line by line (Analyst)
- [economics] What an inference dollar buys in 2026, and where the margin sits (Analyst)
- [economics] Reading an AI company's S-1: the three numbers that matter (Analyst)
- [economics] The cost curve of training vs serving — which one is the business (Analyst)
- [economics] What an inference token actually costs to serve, and how far above
  marginal cost an API price really sits — the markup nobody publishes (Analyst)
- [what-every-engineer-should-know] The capex behind the API: GPUs, power, and
  why the constraint is the grid (Analyst)
- [how-it-works] GPU depreciation: book life vs economic life, and why a levered
  neocloud's debt outlives the asset it's secured against — the schedule mismatch
  that actually decides refinancing risk (Analyst) [surfaced by the 07-12
  circular-financing dive]
- [news-to-framework] Vendor financing, from Lucent to Nvidia: why a supplier
  lends to its own customers, and the two ratios (financing ÷ revenue, top-customer
  concentration) that flip it from a growth lever into a warning sign (Analyst/
  Contrarian) [surfaced by the 07-12 circular-financing dive]
- [how-it-works] How a data-center SPV is actually built: the minority-stake JV,
  the fully-amortizing senior secured note, the operating lease and the guarantee —
  a line-by-line read of the Meta/Blue Owl Hyperion structure and what consolidates
  where (Analyst) [surfaced by the 07-25 off-balance-sheet dive]
- [economics] Private credit is the new bond desk for AI: how Blue Owl / Apollo /
  PIMCO / BlackRock came to hold ~$200B+ of data-center debt, what covenants they
  actually wrote, and why the risk left the public bond market for a channel that
  reports to nobody (Analyst/Contrarian) [surfaced by the 07-25 off-balance-sheet dive]
- [what-every-engineer-should-know] Reading the footnotes: the three lines in a 10-Q
  (commitments-not-yet-commenced, unconsolidated VIEs, guarantees) that carry more of
  a hyperscaler's real AI leverage than its headline balance sheet (Contrarian)
  [surfaced by the 07-25 off-balance-sheet dive]
- [economics] The memory supercycle from the fab up: why HBM's 3:1 (→4:1 at HBM4/4E)
  wafer trade against DDR5 lets AI capex set the price of consumer/prosumer RAM, how
  the DRAM cycle has always worked (glut → margin allocation → shortage), and how to
  read a shortage forecast (RDIMM bit-supply growth vs demand, HBM allocation %)
  before you buy — the deeper standalone follow-on to the 08-23 self-hosting-tax dive
  (Analyst) [surfaced by the 08-23 memory-supercycle dive]
- [x-vs-y] Rent vs own inference, the general rule: the break-even token-volume
  formula (utilization × months-the-model-stays-current × floor price, against capex
  + power), and why model *depreciation* — not the hardware — is the term that decides
  it, so owning wins only where the meter legally/physically can't reach (Analyst)
  [surfaced by the 08-23 memory-supercycle dive; sibling to local-coding 06-17,
  bandwidth-wall 08-04]

## Live — politics & policy (technical lens)

- [news-to-framework] How export controls actually shape which models you can
  run (Contrarian)
- [what-every-engineer-should-know] What the EU AI Act asks of a team shipping
  an agent, in plain terms (Builder)
- [news-to-framework] Provenance and attestation: what "signed software" can and
  can't promise (Analyst)

## Live — cross-domain / investigative (when the story pulls there, 2026-07-04)

*Angles that naturally travel across beats — AI × economy × energy × politics ×
geopolitics — in the mold of the 2026-06-29 silicon dive. Use one when a live
story genuinely leads here; the reader warned against **forcing** the crossover
(especially AI-plus-politics). Pick these because they're the best story that
day, not to hit a cross-domain quota. For the standing devtools / dev-marketing
weekly slot, draw from the "devtools & systems" and "DevRel, dev marketing,
product engineering" pools above.*

- [news-to-framework] Data centers became a ballot issue: the Utah primary
  upset, ~70% local opposition, and the mechanism nobody explains — how a
  hyperscaler load actually lands on a household electric bill (capacity
  markets, interconnection queues, ratepayer cost allocation). AI × energy ×
  local politics × economy (Contrarian/Analyst)
- [economics] The AI trade as a macro variable: how the S-1 wave (Anthropic,
  OpenAI ~$1T) made frontier-model deceleration a thing that moves the Nasdaq —
  what actually transmits from a lab's run-rate to an index (Analyst)
- [news-to-framework] The frontier's guest list: GPT-5.6 Sol shipping to ~20
  government-approved partners as the first real test of state-managed model
  access — the EO mechanism, who's on the list, and what China's clones do to
  the logic (AI × politics × geopolitics) (Contrarian)
- [economics] Bring-your-own-power: why hyperscalers are signing 20-year deals
  with oil majors (Chevron–Microsoft) and 1GW BYOP deals (Vantage–Liberty) —
  the grid is the constraint on inference, and the workaround reshapes energy
  markets (Analyst)
- [news-to-framework] The export-control boomerang: the Fable 5/Mythos ban as a
  case study in a control that taxes the honest domestic lab while the
  capability walks free as open weights — the geopolitics of a leaky border
  (Contrarian)
- [economics] What an inference dollar pays for once you trace it to the
  physical world: chip margin → power contract → water → local tax abatement →
  the ratepayer. Follow one token to the grid (Analyst)
- [news-to-framework] The maintainer revolt meets the labor question: AI-slop
  PRs, review as the scarce resource, and what "generation is free, review is
  charged in social capital" says about who captures the productivity gain
  (Contrarian)
- [news-to-framework] Talent as the real moat: the DeepMind→Anthropic exodus
  ($270B Alphabet wipe) and what a frontier lab's hiring flows reveal about
  where the capability actually lives — people, not weights (Contrarian)

## Used

- [practical-guide] How to size a model-selection eval: the sample-size math for
  telling two close models apart (binomial CI, McNemar on paired items, discordant-pair
  power), how many clean items to resolve a 2-point gap, and why a private post-cutoff
  set beats a saturated public leaderboard (Analyst/Builder) — 2026-08-26 (as "A
  Two-Point Lead Needs Four Thousand Questions to Be Real" by Quist, how-it-works/
  what-every-engineer-should-know; SWE-bench Verified N=500 → single-score 95% CI ±4pt,
  a 2pt gap is a coin flip, ~4,000 paired items needed to resolve it; pegged to Miller/
  Anthropic's error-bars study + SWE-agent run-to-run variance),
  reports/deep-dives/2026-08-26-two-point-lead-eval-sample-size.md
- [news-to-framework] Why training data is the moat and the liability at once —
  copyright exposure as the reason no frontier model ships its corpus (Contrarian)
  — 2026-08-18 (as "The Moat Is Made of the Thing They Can Never Show You" by
  Okafor, pegged to Anthropic's IPO $190–200B 2028 revenue forecast + Bartz v.
  Anthropic's $1.5B settlement; the corpus is the moat MADE OF the liability →
  never disclosed because disclosure is discovery),
  reports/deep-dives/2026-08-18-training-data-moat-and-liability.md
- [x-vs-y] JSON tool calls vs code actions: why writing actions as code composes
  better than emitting tool-name-and-args JSON (Analyst) — 2026-08-16 (written as
  how-it-works/architecture by Vance, pegged to the Mistral "code implemented tool
  calls" patent + the Docker Sandboxes / DeepSeek Harness sandbox wave),
  reports/deep-dives/2026-08-16-agent-writes-code-to-call-tools.md
- [practical-guide] Subagent orchestration that saves wall-clock time, not
  just tokens (Operator) — 2026-07-30 (reframed: subagent = context-isolation
  primitive, not speed; depth + observability),
  reports/deep-dives/2026-07-30-subagents-context-not-speed.md
- [how-it-works] Why AI is better at math counterexamples than proofs: the
  verifier asymmetry — a counterexample is a certificate you check in one pass,
  a proof isn't; search-against-a-cheap-verifier is the shape of every task
  agents win at (Analyst) — 2026-07-24 (as news-to-framework, pegged to Fable's
  Jacobian-conjecture counterexample + Tao/Buzzard),
  reports/deep-dives/2026-07-24-verifier-asymmetry-check-vs-find.md
- [reference] Sandbox mode: OS-level filesystem/network isolation for Bash and
  its children, and when it beats a permission rule you can subvert (Operator) —
  2026-07-23, reports/deep-dives/2026-07-23-sandbox-is-the-real-brake.md
- [how-it-works] How a context window actually works: attention cost, the
  lost-in-the-middle effect, and where recall degrades (Analyst) — 2026-07-21
  (as news-to-framework, pegged to OpenAI's Codex 372k→272k cut),
  reports/deep-dives/2026-07-21-context-window-price-cliff.md
- [practical-guide] Reading your context window: /context, /doctor, and what's
  actually eating your tokens — the fixed preamble + the MCP tool-schema tax
  (Operator) — 2026-07-16,
  reports/deep-dives/2026-07-16-context-tax-before-your-prompt.md
- [what-every-engineer-should-know] What a benchmark number actually measures —
  and how to read a model card without being fooled (Contrarian) — 2026-06-13,
  reports/deep-dives/2026-06-12-reading-a-coding-benchmark.md
- [practical-guide] Designing a subagent fan-out that doesn't blow your token
  budget (Builder) — 2026-06-13,
  reports/deep-dives/2026-06-13-subagent-fan-out-budget.md
- [contrarian] "Open source AI" — what the license actually grants you
  (Contrarian) — 2026-06-16,
  reports/deep-dives/2026-06-16-open-weights-is-not-open-source.md
- [news-to-framework] Can a local model do your daily coding? open-weight vs
  runnable, the memory-budget chain (Builder) — 2026-06-17,
  reports/deep-dives/2026-06-17-local-coding-model-memory-budget.md
- [what-every-engineer-should-know] Prompt caching: what gets cached, the TTL,
  and the cost math that makes or breaks an agent (Analyst) — 2026-06-18,
  reports/deep-dives/2026-06-18-prompt-caching-hit-rate.md
- [reference] What "agent" actually means in 2026, stripped of marketing
  (Contrarian) — 2026-06-19,
  reports/deep-dives/2026-06-19-agent-is-a-control-flow-decision.md
- [how-it-works] How Claude Code's context compaction works, and when it costs
  you (Builder) — 2026-06-20,
  reports/deep-dives/2026-06-20-claude-code-compaction-save-point.md
- [how-it-works] Mixture-of-experts, explained from the routing up (Analyst) —
  2026-06-21, reports/deep-dives/2026-06-21-mixture-of-experts-active-parameters.md
- [how-it-works] Git worktrees under an agent: why isolation beats a shared
  checkout (Builder) — 2026-06-23,
  reports/deep-dives/2026-06-23-git-worktrees-agent-isolation.md
- [how-it-works] How speculative decoding makes models faster without changing
  outputs (Analyst) — 2026-06-24,
  reports/deep-dives/2026-06-24-speculative-decoding-idle-compute.md
- [practical-guide] Context-budget hygiene: keeping a long session from rotting
  before the task is done (Operator) — 2026-06-25,
  reports/deep-dives/2026-06-25-context-budget-sixty-percent.md
- [what-every-engineer-should-know] Idempotency: the one property that saves
  every retry path (Builder) — 2026-06-26,
  reports/deep-dives/2026-06-26-agent-retries-idempotent-writes.md
- [how-it-works] Model distillation: what training on another model's outputs
  actually copies, and what the license lets you do with it (Analyst) —
  2026-06-27, reports/deep-dives/2026-06-27-distillation-without-logits.md
- [news-to-framework] When a model price cut is a moat move, not a gift
  (Contrarian) — 2026-06-28,
  reports/deep-dives/2026-06-28-price-cut-is-a-weapon.md
- [x-vs-y] Long-context vs RAG: when each wins, with real recall numbers
  (Analyst) — 2026-06-30,
  reports/deep-dives/2026-06-30-long-context-vs-rag.md
- [practical-guide] PreToolUse hooks as the security boundary that permission
  patterns can't be (Operator) — 2026-07-02,
  reports/deep-dives/2026-07-02-hooks-are-the-real-guardrail.md
- [how-it-works] How statistical LLM watermarking actually works: green/red-list
  logit bias (Kirchenbauer), the z-test a detector reads, and why paraphrase
  washes it out (Analyst) — 2026-07-03,
  reports/deep-dives/2026-07-03-llm-watermark-paraphrase-ceiling.md
