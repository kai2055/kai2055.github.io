---
title: "BUG-001 — A Threshold Applied Where It Didn't Belong"
description: "A latent bug that stayed invisible until a placeholder config changed from 1.0 to 0.30 — because two different retrieval modes were forced through one code path."
slug: "bug-001-distance-threshold"
date: 2026-06-29
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Bug Report
    - Retrieval
    - Evaluation
---

> **The short version, for anyone:** The system had two different jobs that
> happened to share one piece of code: *ranking* results by closeness, and
> *fetching an exact set* of records that match a filter. A cutoff meant only for
> the first job was silently being applied to the second — but it stayed harmless
> as long as a temporary setting was loose enough to never cut anything. The moment
> that setting was tightened to a realistic value, filter accuracy collapsed from
> 100% to 33%. The bug had been there since the first line of code; changing one
> config value exposed it. The lesson: *tests passing at a permissive setting are
> not proof of correctness.*
>
> *Full bug report below.*

---

| | |
|---|---|
| **ID** | BUG-001 |
| **Component** | `retrieve()` — `src/embedding.py` |
| **Severity** | Major · **Priority** High |
| **Status** | Closed — fixed and verified |
| **Related** | TP-001, ADR-013 |

## What happened

`RELEVANCE_THRESHOLD` was lowered from 1.0 to 0.30. Semantic queries improved.
Filter queries collapsed. Filter precision, recall, and exact-match all dropped from
1.000 to 0.333 — one of three queries passing. The same value across three runs. Not
noise. A logic bug.

The defect was already there. At 1.0, nothing ever got cut, so the bad path never
ran. A config value changed and the latent bug surfaced.

## Evidence

| Metric | t = 1.0 | t = 0.30 | Delta |
|---|---|---|---|
| Hit rate@5 | 1.000 | 1.000 | — |
| MRR | 0.949 | 0.949 | — |
| Section accuracy | 0.435 | 0.435 | — |
| Decline rate | 0.000 | 0.600 | +0.600 |
| **Filter precision / recall / exact** | **1.000** | **0.333** | **−0.667** |

Regression is isolated. Everything else held.

## Root cause

`retrieve()` applied the cosine-distance threshold to *every* query — including
metadata-filtered ones:

```python
# Keep only results close enough to be relevant
relevant = [r for r in results if r["distance"] <= threshold]  # applied unconditionally
```

Two things are wrong. The list comprehension has no condition — every caller gets
the distance cutoff, so semantic ranking and metadata set-retrieval share one path.
And the docstring states it as a flat rule, with no sign that a metadata query might
need different treatment. The defect is in the design, not a slip in the code.

A filter query like "minor-severity incidents" is a **set question**. The right
answer is *every* document matching the filter — it doesn't matter how semantically
close the query phrase sits to the chunk text; the metadata already decided. At 1.0,
nothing was ever discarded, so the wrong logic produced correct output by accident.
At 0.30, documents that matched the filter correctly but sat far in cosine distance
got silently dropped. Filter scoring is exact set-match — lose one document, lose
the query.

**The real defect:** two retrieval modes — semantic ranking and metadata
set-retrieval — forced through one code path, with a parameter meant for one mode
leaking into the other.

## Fix

`retrieve()` now accepts `threshold=None` — no distance cutoff, return all
metadata-matched results, ranked. The call site for filter scoring passes
`threshold=None`; semantic queries keep the real threshold. The two modes are now
distinguished at the call site.

Fixed at the source — `retrieve()`, not the eval harness — so every downstream
caller inherits it, including the Layer 2 diagnostic agent.

## What was missed

`score_filter_query` passes `top_k=5`. If a metadata filter matches more than 5
documents, the vector store returns only the top 5 by distance, and set-match scoring
counts the rest as missing. Harmless now — no filter in the 15-document corpus exceeds
5 — but the same defect class: a semantic parameter constraining a metadata query.
Tracked separately, to fix before the corpus grows.

## Lessons learned

**1. Passing tests at a permissive setting are not evidence of correctness.** The bug
existed from the first line of code. `RELEVANCE_THRESHOLD = 1.0` meant the faulty
branch never discarded anything. The placeholder was even flagged in the source
(`# loose placeholder`). Knowing a setting is temporary is not the same as testing
what happens when it changes.

**2. Partial metric reporting hides regressions.** The threshold sweep tracked 3 of 5
metrics — hit rate, MRR, decline rate. It "confirmed" 0.30 as optimal while silently
breaking a metric it did not watch. Caught only on the confirming run that reported
the full set. Fix: full metrics on every run, enforced by exit criteria.

**3. Determinism separates bugs from noise.** The first thought was jitter. Three
identical runs at 0.333 reclassified it as logic, worth investigating.
