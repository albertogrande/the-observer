# The Box Your Agent Runs In Is a Commodity. The Kit Around It Isn't.

*Deep dive · 2026-08-17 · Docker joined the agent-sandbox war this week. The isolation is the free part — here's what you're actually choosing.*

Your coding agent just ran a command. Maybe it was `pip install` from a lockfile it wrote. Maybe it was a test suite that shells out. Maybe it was `curl` piped to `bash` because a README told it to. Where did that run?

If the answer is "on my laptop, with my SSH keys and my `~/.aws/credentials` one directory up," you already understand why a market erupted this year around a boring-sounding primitive: the disposable box an agent runs code inside. This week, [Docker joined it](https://www.docker.com/products/docker-sandboxes/) — its sandbox product [took Hacker News's #2 slot](https://news.ycombinator.com/item?id=49239751) at 613 points — landing on top of a field already crowded with [e2b, Modal, Daytona, Cloudflare, and Vercel](https://blog.logrocket.com/comparing-ai-agent-sandbox-platforms-e2b-modal-daytona-and-more/). When Docker shows up to a category, the category is real.

Here is the argument of this piece. The sandbox war is not a fight over isolation. Isolation is a solved computer-science problem, and every serious implementation of it is open source and free. The fight is over the layer *above* the box — the environment definition, the persistence, and the integration — because that is the only part with a moat. Choose your sandbox by the shape of your workload, not by the vendor's logo. And respect the one tradeoff no marketing page mentions: the fastest, densest, cheapest boundary is the weakest one, and a security team walked straight out of the popular in-process kind three weeks ago.

## Why the box suddenly matters

For twenty years, sandboxing was a problem you had if you ran *other people's* code — a CI runner, a serverless function, a Figma plugin. The novel thing about 2026 is that the untrusted principal is now your own agent.

A coding agent is a program that writes programs and runs them, in a loop, mostly unattended. It installs packages you didn't vet. It executes scripts it generated from a prompt. And increasingly, *every tool call is becoming code execution*: the pattern we [dived last week](./2026-08-16-agent-writes-code-to-call-tools.md) — the agent writes a snippet that calls your tools, instead of emitting a structured tool-call — turns the model's output directly into something that runs. Cloudflare ships it as "Code Mode." The behavior is convenient and it is also the whole security problem in one sentence: the model's text is now an executable.

The agent doesn't have to be malicious to hurt you. It has to be *wrong*, once, with your filesystem mounted. We've been circling this all summer from different sides: [git worktrees](./2026-06-23-git-worktrees-agent-isolation.md) to stop parallel agents from corrupting each other's files, and [the sandbox as the only real brake](./2026-07-23-sandbox-is-the-real-brake.md) — the observation that a string-matching permission rule can't read intent, but an OS boundary doesn't need to. The sandbox is where that brake actually lives. So the question stopped being "should my agent run in a sandbox" and became "which one, and what am I really buying."

## The spectrum: four boundaries, four tradeoffs

Every product in this market is a point on a single curve: how strong is the wall, versus how fast and cheap is it to stand one up. There are four rungs.

| Boundary | Who ships it | Cold start | Isolation strength | Best for |
|---|---|---|---|---|
| **V8 isolate** (in-process) | Cloudflare Dynamic Workers | single-digit ms | Weakest — one OS process, shared address space | Bursty, short, low-trust-but-not-adversarial JS |
| **gVisor** (user-space kernel) | Modal | tens of ms | Medium — syscalls intercepted, no real kernel | Python/data workloads at density |
| **Container** (shared host kernel) | Daytona, Cloudflare Sandbox | ~90 ms | Medium — namespaces, one kernel to escape | Long-lived, stateful project workspaces |
| **microVM** (dedicated kernel) | e2b (Firecracker), Docker | ~150 ms | Strongest short of a full VM | Untrusted or adversarial code; per-session teardown |

The numbers are real and they cluster. Daytona [cold-starts a container in about 90ms](https://www.zenml.io/blog/e2b-vs-daytona); e2b stands up a Firecracker microVM in about 150ms. Cloudflare's V8 isolates [boot in single-digit milliseconds and cost a few megabytes each](https://www.infoq.com/news/2026/08/cloudflare-computer-agents/) — "roughly 100× faster to boot and 10–100× more memory efficient than typical containers." That density is why Cloudflare can pack thousands of tenants onto one machine.

It is also exactly why the isolate is the weak rung. Which brings us to the part the launch posts skip.

## The tradeoff nobody advertises: cheap is weak

In late July, Check Point Research published [an end-to-end break of Cloudflare's Code Mode](https://research.checkpoint.com/2026/when-agentic-glue-melts/): five memory-corruption bugs in `workerd`, the open-source runtime under Cloudflare Workers, two rated Critical. They chained them into two attacks — one Worker reading another tenant's API tokens out of a shared heap, and a full sandbox escape to native code execution on the host.

The root cause is architectural, not a slipped patch. Workerd "packs many customers into a single OS process, using V8 isolates as the security boundary." Check Point states the tradeoff plainly: "Isolating untrusted code without an OS-level boundary means no VM, no container, just a V8 isolate inside a shared process. That puts the entire security model on a single software boundary." One bug in that boundary and every tenant in the process is exposed. Cloudflare fixed it in production, fast. The lesson survives the fix: the rung that boots in single-digit milliseconds is the rung with the least between your agent and the host.

So the curve is honest. You do not get microVM isolation at isolate latency. Anyone who implies otherwise is selling you the boot time and hoping you don't ask about the wall.

The deciding quantity, then, is not "which sandbox is best." It's **how adversarial is the code inside?** There are two regimes, and they want different rungs:

- **Untrusted-but-not-adversarial.** The model wrote the code, against your own repo, to do a task you asked for. The failure you're guarding against is a mistake — a bad `rm`, a runaway install, a leaked secret from an over-broad mount. A container is plenty. You're building a blast wall, not a prison.
- **Genuinely adversarial.** The code came from the open internet, or a third-party MCP server, or another tenant, or a prompt-injected instruction the agent obeyed. Now the code inside is *trying* to get out. This is the regime that needs a kernel boundary — a microVM — and it's worth the 150ms.

Most working engineers are in the first regime and reach, correctly, for a container. The mistake is running the *second* regime's workload on the first regime's wall because the demo booted faster.

## Why nobody wins by having the best box

Here's the uncomfortable part for the vendors. The isolation itself is free.

Firecracker — the microVM monitor under e2b and half the serverless world — is [open source from AWS](https://github.com/firecracker-microvm/firecracker), and has been for years. [gVisor](https://gvisor.dev/) is open source from Google. V8 isolates are open source from the Chrome team. The hard cryptographic-grade CS of "run untrusted code without letting it touch the host" was solved by hyperscalers and given away. Docker's one genuine engineering choice this week was to build a *proprietary* VMM instead of using Firecracker — not for a better wall, but so the microVM [runs on macOS and Windows](https://collabnix.com/whats-new-in-docker-in-2026-sandboxes-hardened-images-and-the-ai-native-container-platform/), where a developer's laptop actually is. That's a portability feature, not an isolation one.

When the core primitive is free, the price collapses to the cost of the compute, and the pricing sheets prove it. Metered by the vCPU-second and the gigabyte-second, the whole field sits inside a narrow band: [e2b runs about $0.166 per sandbox-hour, Daytona matches it, Modal is about $0.238](https://blog.logrocket.com/comparing-ai-agent-sandbox-platforms-e2b-modal-daytona-and-more/) — a spread of maybe 1.5×, with free tiers ($100–$200 in credits) papering over even that. This is electricity pricing. You do not build a durable business selling a commodity within 1.5× of every competitor. If your team has steady, high volume, [self-hosting Firecracker crosses under the managed price](https://bex.co/blog/2026/08/07/self-hosting-firecracker-sandboxes-ai-agents-cost-crossover) at exactly the point you'd expect from a rented commodity — which is the tell that it *is* one.

We have watched this movie all year, one layer at a time. The model [commoditized](../2026-W33.md). The harness commoditized. The [agent-instructions file commoditized](./2026-08-07-agents-md-rules-travel-guardrails-dont.md). Now the execution box. Each time, the open-source version of the primitive shows up and the price goes to the floor, and each time the same question follows: if not the primitive, then where does the value sit?

## The moat is the kit, not the box

It sits in the layer above the wall — the part that isn't the isolation.

Look at what each vendor actually asks you to write down. Docker's [Sandbox Kits](https://www.docker.com/products/docker-sandboxes/) are a YAML spec: the tools, the environment variables, the injected credentials, the allowed network domains, the files to drop in, the startup commands. e2b has templates. Daytona sells [persistence and snapshots](https://www.zenml.io/blog/e2b-vs-daytona) — a stateful workspace the agent returns to. Cloudflare sells [integration](https://www.infoq.com/news/2026/08/cloudflare-computer-agents/) — one persistent agent workspace routed across isolates, containers, and its edge network, so the sandbox sits next to your KV store and your queues.

None of that is isolation. All of it is switching cost. Your environment definition is the thing you'd have to rewrite to move vendors; the raw microVM underneath is fungible. This is the same law we keep finding: the primitive commoditizes, and the value accrues to the configuration and wiring layered on top — exactly what we found with [enforcement config](./2026-08-07-agents-md-rules-travel-guardrails-dont.md) (rules travel between harnesses, guardrails don't) and with [the channel](./2026-06-09-channel-was-the-product.md) more broadly (the moat is distribution, not the artifact). The environment off-ramp the channel thesis named a year ago just grew a product category.

And note who else is building sandboxes: the labs themselves. Anthropic ships one; OpenAI's Codex runs in one. When the model vendor also owns the box the model's code runs in, that's the same vertical-integration reflex we tracked in the pricing thread — own the whole stack from weights to execution so the customer never has to leave. The sandbox is one more surface in that land grab.

## What to do Monday morning

You run a coding agent every day. Four moves.

**Match the wall to the code, not to the benchmark.** If your agent writes code against your own repo, you're in the untrusted-but-not-adversarial regime — a container (Daytona-class, or Docker's local microVM if you want the stronger wall on your laptop) is right, and the 90ms cold start is invisible. Reserve microVM-per-session and pay the latency only when the code inside is genuinely hostile: arbitrary third-party execution, untrusted MCP servers, anything internet-fetched, or multi-tenant.

**Never run adversarial code on an in-process boundary.** V8 isolates are a beautiful fit for bursty, cheap, low-trust JavaScript at scale. They are the wrong tier for "the agent obeyed a prompt injection and is now trying to exfiltrate," and Check Point just demonstrated why. Fast boot is not a security property.

**Own the kit; rent the box.** Keep your environment definition — the Dockerfile, the YAML, the setup script — in your own repo, portable and vendor-neutral. The isolation underneath is a commodity priced like electricity, so treat it like one: keep the ability to switch. The thing you must not lose is the definition of your environment, because that's the part with the switching cost the vendors are counting on.

**Scope the mounts and the network before you scope anything else.** The most common real incident isn't a kernel escape; it's a sandbox that was handed the whole home directory and an open network. The strongest wall around an over-broad mount still leaks. Start from nothing mounted and no egress, then add back exactly what the task needs — which is what [Sandbox Kits](https://www.docker.com/products/docker-sandboxes/) and their equivalents are for.

## What would change this read

The thesis is that isolation is a commodity and the value lives in the kit. Two things would break it.

If a vendor ships an isolation boundary that is genuinely both fast *and* strong — say, a hardware-backed enclave that boots like an isolate but walls like a microVM, and that competitors can't clone from open source — then the box stops being fungible and the moat moves back down to the wall. Watch the confidential-computing and microVM-snapshot research for that; it isn't here yet.

And if the price band *widens* instead of holding — if one vendor sustains a 3–5× premium and keeps its customers — then something other than compute is being sold, and I'm wrong that this is electricity. So far the band is tight and the primitives are all open, so the deciding quantity stays: not who has the best box, but who owns the environment definition when the box underneath costs the same everywhere.

The agent needs somewhere to run its code. This week made "somewhere" a real market. Just remember that the market is selling you the kit and the convenience — the wall itself was free, and open, before any of them showed up.
