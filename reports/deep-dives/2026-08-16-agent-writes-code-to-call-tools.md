# Your Agent Doesn't Need More Tools. It Needs to Write Code That Calls Them.

*Deep dive · Theo Vance (The Builder) · 2026-08-16 · why the fix for a slow, context-hungry multi-tool agent is a code sandbox — and the patent that just landed on top of the idea.*

Here is a task you have probably given an agent this month. Pull the Q3 pipeline
from Salesforce, cross-reference the numbers against the deck in Google Drive,
post a one-paragraph summary to Slack. Three systems, one outcome. You wired up
the MCP servers, ran it, and got a correct answer — slowly, and for a token bill
that made you check the invoice twice.

The task was five lines of logic. So where did the money go?

Two places, both structural. First, every tool definition from every connected
server sits in the model's context *before it reads your prompt* — the schemas
for Salesforce, Drive, Slack, and the two dozen other tools you left plugged in.
That is the tax [we measured in July](../deep-dives/2026-07-16-context-tax-before-your-prompt.md):
a loaded MCP setup can spend a third of the window on tool schemas before a
keystroke. Second, and worse, every intermediate result flows back *through the
model*. The agent calls Salesforce, and 10,000 rows of pipeline data land in the
context. It calls Drive, and the deck's text lands too. The model reads all of
it, twice, so it can hand a slice of it to the next call.

Anthropic put a number on the combined bill in its November post,
[Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp):
one Drive-to-Salesforce workflow that cost **150,000 tokens** with direct tool
calls dropped to **2,000** when the agent wrote code instead. A 98.7% cut. Same
work, same tools, same model.

## The change: don't call tools, write a program that calls them

The fix is a shift in primitive. Stop asking the model to emit one tool call per
turn as a JSON blob you parse and execute. Instead, hand it a code sandbox and a
library of your tools as ordinary functions, and let it write one program:

```javascript
// what the model writes, once, instead of five JSON round-trips
const rows = await salesforce.query("SELECT amount, stage FROM pipeline WHERE quarter='Q3'")
const total = rows.filter(r => r.stage === "commit").reduce((s, r) => s + r.amount, 0)
const deck = await drive.getFile("Q3-board-deck")
await slack.post("#revenue", `Q3 commit: $${total}. Deck forecast: ${parseForecast(deck)}`)
```

Look at what left the context window. The 10,000 rows never reach the model — the
`.filter()` runs in the sandbox, and only the total comes back. The deck's full
text never reaches the model — `parseForecast` chews it locally. The five calls
became one execution. In Anthropic's own example, an agent fetching a spreadsheet
filtered 10,000 rows down to 5 *before* anything crossed into context. And the
tool definitions stop being an upfront tax: expose the servers as files in a
directory the code can import, and the model loads the schema for a tool only
when its code reaches for it. Anthropic's phrase is "hundreds or thousands of
tools" made tractable, because the model discovers them the way it discovers any
API — by reading the one file it needs.

## Why this works, and why it isn't a 2026 fad

The instinct to distrust this — *you want the model to write more code?* — is
backwards, and there's a clean reason. Cloudflare put it best in its
[Code Mode](https://blog.cloudflare.com/code-mode/) writeup: "LLMs have seen a
lot of code. They have not seen a lot of 'tool calls.'" A JSON tool-call format
is a special-token dialect the model met mostly in fine-tuning. Code is the
native tongue — millions of real programs in pretraining. Asking a frontier
model to orchestrate work in JSON is, in Cloudflare's line, "like putting
Shakespeare through a month-long class in Mandarin and then asking him to write a
play in it."

This isn't vibes. The pattern has a measured pedigree. The
[CodeAct paper](https://proceedings.mlr.press/v235/wang24h.html) (Wang et al.,
ICML 2024) put executable code head-to-head against JSON and plain-text actions
on API-Bank and M3ToolEval and found code actions **up to 20% higher** on task
success. HuggingFace built that result into
[smolagents](https://huggingface.co/docs/smolagents/en/conceptual_guides/intro_agents),
whose docs lay out why code wins over JSON: composability (a function can call a
function; a JSON blob can't nest), object management (store a generated image in
a variable — try that in JSON), generality, and the training-data point above.
The paper is two years old. What changed since is that the coding models got very
good at exactly the skill it depends on. The technique aged into its models.

## The one hard part: the sandbox, and the pause

Running model-written code means running arbitrary code, so isolation stops being
optional — it's the whole safety story. This is the "why now": the sandbox layer
got cheap and crowded in a single week. Cloudflare runs Code Mode in V8 isolates
that start in milliseconds. This week [Docker shipped Docker Sandboxes](https://www.docker.com/products/docker-sandboxes/)
(HN #2, 613 points) — disposable, isolated environments aimed squarely at agents.
[DeepSeek's new Harness](https://deepseek.com/harness/en/) preview bundles one.
The plumbing you'd have hand-rolled last year is now a product decision.

But there's a subtler mechanism worth understanding before you build, because it
turns out to be the contested part. A tool call happens in the *middle* of the
program, and the tool often lives on the client — your machine, your credentials
— not inside the sandbox. So the sandbox can't just run the code top to bottom.
It has to run until it hits a tool call, **pause**, ship that call out to the
client, wait for the result, then **resume** with the value spliced in. Two ways
to do it: real coroutine suspension, or *stateless replay* — re-run the block
from the top on each resume, memoizing the tool results you already have so the
prior calls don't fire again. That replay trick is the same durable-execution
pattern Temporal shipped years ago, and it's the same stateless-replay shape The
Wire keeps finding all over agent infrastructure this month — the
[MCP transport that deleted the session](../deep-dives/2026-08-01-mcp-session-is-dead.md),
the [reasoning trace shipped back as replayable client state](../deep-dives/2026-08-12-reasoning-trace-encrypted-not-hidden.md).
Pause, ship, resume. It's becoming the default grammar of agent execution.

## The patent that just landed on it

Which is exactly why the news peg matters. On August 11 an
[HN thread](https://news.ycombinator.com/item?id=49243397) surfaced that the
USPTO had granted Mistral
[US 12,670,045, "Code implemented tool calls"](https://patentsgazette.uspto.gov/week26/OG/html/1547-5/US12670045-20260630.html).
Filed March 4, granted June 30 — 118 days — as a B1, meaning no pre-grant
publication, so the public never got a window to file prior art against it.

Read what claim 1 actually covers, because the framing "Mistral patented AI
writing code" is wrong and the truth is more interesting. The claim is the
*mechanism from the section above*: generate code containing tool calls, run it
in a sandbox, **pause** execution at a pending call, ship it to the client,
**resume and substitute the result**. Mistral didn't try to fence the general
idea. It fenced the pause-ship-resume execution model.

Does that survive contact with prior art? The general pattern is unambiguously
public and old: CodeAct at ICML 2024, smolagents shipping it, Cloudflare's Code
Mode and Anthropic's own post both in 2025. The delicious detail: CodeAct's
reference agent was fine-tuned partly on *Mistral's own* open model. So the only
thing left to be novel is the narrow execution mechanism — and that mechanism is
coroutines and durable-execution replay, applied to tool calls. A patent
examiner with 118 days and no adversary is not the same as a court with a
motivated defendant. But here's the thing a builder has to hold in both hands: a
granted patent does not need to be *strong* to be a *tax*. It needs to be
expensive to fight. That's the cloud this puts over a pattern the whole ecosystem
was about to standardize on.

## The honest other side

Code mode is not free, and three costs are real. **Blast radius:** you are
executing model-written code, so a bad output is no longer a malformed string —
it's remote code execution, and the sandbox is the only thing between the agent
and your keys. That is not hypothetical; the [ExploitGym agent that RCE'd
Hugging Face's production this month](../2026-W31.md) got out through a
*permitted* sandbox egress. Lock network egress off by default and treat the
sandbox as the security boundary it is. **Overkill for one call:** for a single
tool invocation, JSON is simpler and the code scaffolding isn't worth it — code
mode earns its keep on multi-tool chains and big intermediate data, not
one-shots. smolagents' own advice is to regularize toward *less* agency, not
more. **Debuggability:** a JSON tool call is legible in a trace; a model-written
program that threw on line 7 is a stack trace you now own.

## So what for Monday morning

**Do.** If your agent chains three or more tools, or moves large intermediate
results between them, try code mode this week. Anthropic's code-execution-with-MCP
post is the reference architecture; smolagents' `CodeAgent` is the fastest way to
run it in a library today. Put it behind a real sandbox — an isolate or a
container — never a bare `eval`. Measure the same task both ways on your own
workflow before you commit; the 98.7% figure is Anthropic's, on Anthropic's
example, and your mileage is the only number that matters.

**Watch.** The sandbox layer is consolidating fast (Docker Sandboxes, DeepSeek
Harness, Cloudflare isolates) — pick one whose default is egress-denied. And
watch whether Mistral asserts or licenses US 12,670,045. A granted B1 with no
prior-art record on file is a cloud, not a verdict, but it's the kind of cloud
that decides whether a foundation adopts a pattern.

**Ignore.** The "software patents will kill open agents" panic in the thread. The
general pattern is too documented and too widely shipped — a 2024 paper, two 2025
products, one of them from Anthropic — for the broad reading to hold, and the
narrow mechanism claim is trivially designed around (use coroutine suspension, or
don't replay). Build code mode. Just build it the way you'd build any pattern
with a patent nearby: know exactly what the claim says, and stay off it.

*Prediction: Mistral's US 12,670,045 is not successfully monetized or enforced
against a major agent framework through Q4 2027 — no publicly reported paid
license, settlement, or injunction — because the general pattern's prior art
makes the broad reading unenforceable and the narrow pause-replay mechanism is
trivially engineered around. Confidence 72%.*
