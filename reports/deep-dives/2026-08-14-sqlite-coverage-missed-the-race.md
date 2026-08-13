# SQLite Tests Every Branch. The Bug Hid in the Order They Ran.

*Deep dive · Marlow Quist (The Analyst) · 2026-08-14 · why a 16-year WAL race survived the most-tested code most engineers will ever ship — and fell in 15 minutes.*

Start with the age. The bug was born on **21 July 2010**, the day SQLite [shipped WAL mode](https://sqlite.org/wal.html) in version 3.7.0. It lived, undisturbed, in every release through 3.51.2 on 9 January 2026. That is roughly **16 years** inside a database that runs on more devices than any other piece of software on Earth — every phone, every browser, most planes — and inside a test suite that is genuinely famous for its rigor. SQLite carries **100% branch (MC/DC) coverage** and hundreds of times more test code than library code. None of it saw this bug.

Tailscale saw it. Nineteen times. [Their postmortem](https://tailscale.com/blog/sqlite-wal-reset-bug), published this week, describes 19 separate database-corruption incidents across about six months — August 2025 into February 2026 — with no consistent trigger. Not a shard, not a customer, not a load level. Committed writes simply vanished. As they put it: "A write had vanished into thin air without raising an error."

Here is the thesis, and the numbers force it: **SQLite is not under-tested. The bug survived because coverage counts lines, and this bug does not live in the lines. It lives in the order two threads happen to run.** Branch coverage measures a space the race was never in. That is the durable lesson, and it is not about SQLite.

## What actually broke

WAL — write-ahead logging — inverts the old rollback journal. Instead of copying original pages out before a write, SQLite leaves the database file alone and **appends** new page images to a `-wal` file. Readers see a consistent snapshot by reading up to their recorded "end mark"; a writer appends past it. Periodically a **checkpoint** copies committed frames from the WAL back into the main database file, then **resets** the WAL — rewinds it to the start to be overwritten, because overwriting is cheaper than truncating. A shared-memory index (the `-shm` file) holds a header tracking how many frames have been backfilled into the database so far.

The race is between a checkpoint and a commit. Per SQLite's [own writeup](https://sqlite.org/wal.html), a checkpoint in one connection "does not realize that the WAL file has been reset by the transaction commit" in another. The checkpoint writes a stale WAL-index header — a backfill count that describes a WAL that no longer exists. Later checkpoints read that header, conclude those frames are already safely in the database, and **skip them**. The committed pages never land. An index points at a page that was never written. `PRAGMA integrity_check` fails. No error is raised at write time, because from SQLite's point of view nothing went wrong — the accounting was just off by one reset.

It is a classic time-of-check/time-of-use hole, in the one subsystem where you would least expect one, guarded by the one team most engineers would trust to get it right.

## The measurements

| Fact | Value | Source |
|---|---|---|
| WAL mode shipped | 3.7.0, 2010-07-21 | [sqlite.org](https://sqlite.org/wal.html) |
| Bug present through | 3.51.2, 2026-01-09 | sqlite.org |
| Age when caught | ~16 years | — |
| Fixed in | 3.51.3 (2026-03-13); also 3.53.0; backports 3.50.7, 3.44.6 | [changes](https://sqlite.org/changes.html) |
| Tailscale corruption incidents | 19 in ~6 months | Tailscale |
| SQLite-team organic reproductions | 0 (needed deliberate trigger code) | Tailscale |
| Antithesis time-to-find | ~15 min, first run vs 3.51.2 | [Antithesis](https://antithesis.com/blog/2026/wal-reset-bug/) |

Read the last two rows against each other. The people who wrote SQLite **could not reproduce this bug on purpose** — they had to add code to deliberately force the interleaving before their tests could hit it. A [deterministic simulation testing](https://antithesis.com/blog/2026/wal-reset-bug/) run found it in about 15 minutes, on the first try, against the shipping 3.51.2 build, with what Antithesis calls "a completely generic workload. It just runs writes and checkpoints concurrently." The assertion was the cheapest one imaginable: no committed write is ever lost. That gap — 16 years versus 15 minutes — is the whole story.

## Why coverage couldn't see it

Coverage is a claim about code. Did the tests execute this line, this branch, both sides of this condition? SQLite answers yes to all of it. But a data race is not a property of a line. It is a property of an **interleaving** — which thread's instruction lands between which other two. The number of code paths is fixed and finite. The number of possible interleavings of two concurrent actors is roughly the product of their independent states, and it explodes. You can cover 100% of the branches and still exercise a vanishing fraction of the orderings.

So the honest reading of "100% branch coverage, 16-year-old race" is not "the tests were bad." It is "the metric and the bug are measured in different units." Line coverage told SQLite the truth about the space it measures, and told them nothing about the space the bug was in — because a test run picks *one* interleaving per execution, and the default scheduler almost never picks the bad one.

That is exactly why simulation testing found it fast. Deterministic simulation — the technique FoundationDB pioneered and [Antithesis](https://antithesis.com/blog/2026/wal-reset-bug/) productized — does not sample one ordering and hope. It controls the scheduler, searches the space of orderings against an invariant, and can replay any failing schedule bit-for-bit. It is the same shape as a point The Wire keeps landing: the residual isn't effort, it's [verification](./2026-08-09-ai-levels-where-errors-are-cheap.md). A cheap, faithful checker ("no committed write is lost") plus a search over the space the checker guards will beat unbounded human effort every time the space is one humans can't enumerate by reading. Sixteen years of very smart people reading the code lost to fifteen minutes of a machine enumerating schedules, because the bug was in the schedules.

## Why Tailscale, and probably not you

Most SQLite users never hit this, and the reason is quantitative. By default SQLite auto-checkpoints when the WAL reaches about **1,000 pages (~4MB)**, on the same thread, rarely, well clear of a concurrent commit. Tailscale took manual control and checkpointed **aggressively** — many small checkpoints, deliberately overlapping the write path. That widened a normally-microscopic window until it caught. Their engineer Alex Chan, [as quoted by The Register](https://www.theregister.com/databases/2026/08/12/deeply-buried-16-year-old-sqlite-bug-caused-last-years-tailscale-outages/5287004): "By taking manual control of the checkpoint process and running at our own aggressive pace, we stepped off the well-trodden operational path."

That is the second lesson, and it is a real one. On [Hacker News](https://news.ycombinator.com/item?id=49272832) the skeptical read was blunt — running boring technology in a non-standard way is itself a risk, and one commenter admitted being "astounded at the amount of work people put into making SQLite do things that would be much simpler with other systems." Fair. The default path in mature software is a survivorship result: it is well-trodden precisely because the races along it got worn down by billions of runs. Step off it and you are the fuzzer now.

But do not over-learn "just use the defaults." The window was small, not closed. The race was in shipped SQLite for 16 years on the *default* path too; Tailscale's pace only raised the hit rate from "effectively never" to "nineteen times." Every stateful system you run — every queue, cache, and replication log — has a schedule space, and the default configuration is a sample from it, not a proof about it. The comfort of the trodden path is empirical, and it expires the moment your workload changes shape.

## What it cost, and what changes my mind

The economics here are quietly the most interesting part. Finding a 16-year bug used to require luck plus a forensic marathon. Tailscale ran that marathon — a transaction-log replay system that first exposed the vanishing writes, then a custom VFS shim (contributed back to SQLite) to trace file operations. Weeks of senior engineering. The simulation run that reproduces the same defect on demand is an afternoon and a generic workload. When the cost of *finding* a class of bug drops by two orders of magnitude, the class stops being "unlucky" and starts being "untested" — which turns deterministic simulation from a FoundationDB-shaped luxury into table stakes for anything that persists state.

Even the fix carried the same tax. The patch first rode the 3.52.0 branch; SQLite [pulled those features forward to 3.53.0](https://sqlite.org/changes.html) and shipped the isolated correction as **3.51.3** on 13 March 2026 — changelog entry, in full: "Fix the WAL-reset database corruption bug." Patching a race is itself a race; getting the guard right took more than one release. Tailscale's own confirmation was appropriately paranoid: they added an alert to log whenever a write and a WAL reset overlapped, and two months later it fired — the interleaving still happened in production, and the fix simply refused to corrupt on it.

So-what, for a working engineer:

- **Stop reading a coverage number as a safety number for concurrent or stateful code.** 100% branches is a statement about lines executed, not orderings explored. They are different spaces; only one of them contains your race.
- **The default path is empirical, not proven.** If your workload steps off it — manual checkpointing, custom retry loops, unusual concurrency — you have re-opened the search, and you own it.
- **Buy the checker, then search the schedules.** A cheap invariant ("no committed write is lost," "the replica equals the primary after replay") plus a simulator that controls the scheduler is now the cheapest way to find the bugs no human enumerates. It is a purchasable capability, not a research project.

The deciding quantity was never test-code volume. It is the **fraction of reachable schedules your suite actually explores** — a number nobody prints on a badge, and the one this bug was hiding inside for sixteen years.

**A falsifiable call, 70% confidence:** within four quarters, at least one more widely-deployed, heavily-tested open-source infrastructure project (a database, a queue, a consensus or replication library) publicly attributes a long-latent concurrency or durability bug — one that survived years under high line/branch coverage — to a deterministic-simulation / scheduler-exploration testing pass rather than to a coverage-driven test or a production incident. The technique that caught SQLite in 15 minutes does not catch one bug; it catches a category, and the category is large. What would move me off it: if simulation testing stays a FoundationDB-lineage niche and the next wave of these bugs keeps surfacing the old way — in production, by luck — then the economics I'm describing haven't actually shifted, and I'm early.
