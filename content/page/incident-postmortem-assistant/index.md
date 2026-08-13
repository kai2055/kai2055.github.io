---
title: "Incident Post-Mortem Retrieval Assistant"
description: "When a system crashes at 3 a.m., it answers the one question that matters: have we seen this before?"
slug: "incident-postmortem-assistant"
date: 2026-02-14
layout: "page"
---

**What breaks without it →** A payment system goes down at 3 a.m. The on-call
engineer has a vague memory that something like this happened last year, but the
write-up is buried in a wiki nobody can search under pressure. So they start
diagnosing from scratch — chasing the wrong lead, waking up colleagues — while
every extra minute of downtime costs revenue and trust. The company already
*paid* to learn this lesson once; now it's paying again because the lesson wasn't
findable.

**What this delivers →** The engineer types what they're seeing in plain words —
"checkout is timing out, database connections maxed" — and the system hands back
the closest past incidents, separating the actual *root cause* from the symptoms,
with every answer traceable to a real documented incident. When nothing genuinely
matches, it refuses rather than guessing. Institutional memory becomes something
you can *query in the moment you need it*, not archaeology you do afterward.

## The problem, in plain terms

Post-mortems are where a company writes down the answer to "has this happened
before?" — but they're unsearchable exactly when you need them, mid-outage. Making
them searchable is the easy half. The hard half — and the reason this project
exists — is **trust**: retrieving the *wrong* past incident during a live outage
actively sends an engineer chasing the wrong root cause. So this isn't just a
search tool. It's a search tool that **measures whether its own answers can be
trusted**, and refuses to answer when they can't.

## What I built

A three-layer system, where each layer answers a different question:

- **Layer 1 — Does it find the right incident?** The core retrieval path: it
  breaks past post-mortems into meaningful sections, converts each into a form the
  computer can match on *meaning* (not just keywords), stores them, and pulls the
  closest matches to a query. It only answers from what it actually retrieved, with
  numbered citations — and declines when nothing is close enough.
- **Layer 2 — What's actually the root cause?** A single search can't tell a cause
  from its side-effects. This layer is a small reasoning agent that breaks an
  incident into separate symptoms, searches for each, weighs the evidence, and
  produces a *ranked* diagnosis with confidence levels. A safeguard strips out any
  incident the model "remembered" but didn't actually retrieve — so it can't cite
  something that isn't there.
- **Layer 3 — Is it *still* as good as it was?** This is the layer most projects
  like this don't have. Every code change automatically re-runs the full quality
  test suite and compares against a saved baseline. **If answer quality drops, the
  change is blocked from merging.** Reliability isn't checked once at launch — it's
  enforced on every single update.

## The results (measured, not claimed)

The evaluation suite is 39 real test queries. At the chosen setting:

- **Hit rate 1.00** — the correct past incident is retrieved *every single time*.
- **MRR 0.92** — and it's almost always ranked first, not buried in the list.
- **Metadata filters: 1.00 precision and recall** — asking for "minor-severity
  config errors at Cloudflare" returns exactly the right set, nothing extra.

And it holds up as questions get harder — even with deliberately tricky,
distractor-filled queries, the right incident is *never lost*, it just ranks
slightly lower:

| Difficulty | Hit rate | Ranked first (MRR) |
|---|---|---|
| Easy (direct) | 1.00 | 1.00 |
| Medium (described by symptom) | 1.00 | 0.96 |
| Hard (with distractors) | 1.00 | 0.82 |

**Why one specific setting?** The retrieval cutoff (0.30) wasn't guessed — it was
chosen from a sweep of options. It's the *lowest* threshold where every real query
still finds its incident, while rejecting the most off-topic junk. Loosen it and
the system starts confidently answering questions it should refuse; tighten it and
it starts missing real incidents. That trade-off is documented with the evidence
behind it — see the deep dives below.

## A note on honesty

This is a reliability project, so I document where it *fails* as carefully as
where it succeeds:

- **It matches on vocabulary, not underlying mechanism.** A couple of "should have
  refused" queries slipped through because they share words with real incidents (a
  DDoS question pulled a Cloudflare outage on shared networking vocabulary).
  Documented as a known failure pattern — not silently patched over.
- **It over-refuses on very terse symptoms.** If a symptom is phrased as bare
  effect ("nothing can log in"), the agent sometimes finds nothing and declines
  even when a match exists.
- **It pinpoints the right *document* reliably, the right *section* about half the
  time** — great for surfacing the incident, weaker for jumping to the exact
  paragraph.

Naming these precisely *is* the reliability work. A system you can't describe the
limits of is a system you don't actually understand.

## Key decisions (the "why," not just the "what")

Every significant choice is an Architecture Decision Record, including the bugs
found along the way — a stale database masquerading as a quality regression,
non-reproducible baselines, fabricated citations slipping past a naive first
check. Two that capture the philosophy:

- **The evaluation framework is the core, not an add-on** — the whole project is
  organized around measuring trust, because an unmeasured RAG system is just a
  confident guess generator.
- **Gate by consequence, not by number** — the CI quality gate blocks hard on the
  failures that would mislead an engineer (fabricated citations, missed incidents),
  while tolerating tiny statistical noise on metrics where noise and signal are the
  same size.

## Engineering evidence — deep dives

The write-ups below are where the reliability claims are actually earned. Each has
a plain-language summary up top; the technical investigation follows. Skip freely.

**Bug reports & investigations**

- [Chasing a Bug That Didn't Exist](/p/chasing-a-bug-that-didnt-exist/) — the code
  was correct and the system was still useless; the eval suite had been grading
  itself on easy questions.
- [The Test Suite That Was Too Slow To Run](/p/test-suite-too-slow/) — 15m 46s →
  8.5s. Correct isn't the same as usable.
- [BUG-001: a threshold applied where it didn't belong](/p/bug-001-distance-threshold/)
  — a latent bug that only surfaced when a placeholder config value changed.

**Findings (the reliability thinking, shown)**

- [A stale vector store caused a phantom regression](/p/stale-vector-store-regression/)
- [Measuring against live vs. frozen inputs](/p/live-vs-frozen-decompose/) — why a
  quality gate must be reproducible before it can gate anything.
- [Confident matches on vocabulary, not mechanism](/p/vocabulary-not-mechanism/) —
  a known failure kept visible on purpose.
- [The Layer 2 baseline, and a grounding filter that leaked](/p/layer2-baseline-grounding-filter/)
- [Why Layer 2 needed its own threshold](/p/layer2-threshold/)

## What this demonstrates

The ability to build an AI system that's *honest about its own confidence* —
retrieval, agent-based reasoning, and, most importantly, a measurement-and-
regression discipline that keeps quality from silently rotting. This is exactly the
gap between "I built a RAG demo" and "I built a RAG system you could actually
depend on."

→ **[GitHub repo](https://github.com/kai2055/incident-postmortem-assistant)**
