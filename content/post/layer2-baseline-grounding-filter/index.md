---
title: "The Layer 2 Baseline, and a Grounding Filter That Leaked"
description: "Building the first measured baseline for the diagnostic agent caught a real bug 30 unit tests had missed: a safeguard that let fabricated citations through as long as one real one rode alongside."
slug: "layer2-baseline-grounding-filter"
date: 2026-07-24
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Finding
    - Evaluation
    - RAG
---

> **The short version, for anyone:** Before you can detect whether a system is
> *getting worse*, you need a measured snapshot of how good it is *now* — a baseline.
> Building that baseline for the diagnostic agent immediately caught a real bug that
> 30 existing unit tests had missed: a safeguard meant to strip out "made-up"
> citations was letting them through, as long as at least one *real* citation rode
> alongside. One genuine reference could smuggle in any number of invented ones. The
> lesson: *unit tests check the cases you thought of; measuring against real output
> catches the ones you didn't.*
>
> *Full finding below.*

---

**Measured on:** 15 descriptions through the full agent · **Result:** first measured
Layer 2 baseline; one grounding bug found and fixed.

## Why this matters

Layer 2 had 30 unit tests proving the machinery worked, and one integration run
judged by eye. It had no numbers. You cannot build a regression gate — which detects
regressions by comparing against a baseline — without a baseline to compare to. This
is that baseline. Building it also caught a real bug the unit tests had missed.

## The bug the baseline caught

The first run reported **2 grounding violations** — candidates citing incident IDs
that were never retrieved. That number is supposed to be impossible by construction,
so the tripwire fired.

### Why it slipped through

The grounding filter kept a candidate if **any** cited ID was real:

```python
cited = {...}                 # every id the model cited
if cited & valid_ids:         # at least one is real?
    grounded.append(d)        # keep the whole thing, evidence unchanged
```

So a candidate citing `aws-s3-2017-02-28, gitlab-2017-01-31` passed — aws-s3 is real,
the intersection is non-empty — and carried the fabricated gitlab citation through
untouched. **The filter checked that at least one citation was grounded. It never
checked that every citation was.** One real id smuggled in any number of invented
ones.

### The fix

```python
real = cited & valid_ids
if not real:
    continue                          # nothing grounds this, drop it
d["evidence"] = ", ".join(sorted(real))  # keep only the real citations
grounded.append(d)
```

A candidate now survives only if it has a real citation, and its evidence is
rewritten to contain *only* real citations. Fabricated ids are stripped, not
tolerated. Covered by a new test that feeds one real and one fake id and asserts only
the real one remains.

### Why the unit tests missed it

The existing grounding test used candidates that were *entirely* fabricated — those
were correctly dropped. The gap was the *mixed* case: one real citation plus one
fake. No test exercised it, so nothing failed. **The scoring run on real model output
was the first thing to hit it. That's the point of scoring against real output: unit
tests check the cases you thought of, evaluation catches the ones you didn't.**

## The baseline (after the fix)

| Metric | Value |
|---|---|
| top-1 accuracy | 0.625 (5 of 8 with a primary cause) |
| any-hit rate | 0.615 |
| noise rate | 0.444 |
| decline rate | 1.000 (2 of 2 no-match entries) |
| mean candidates | 1.80 |
| grounding violations | 0 |
| mean iterations | 2.20 |

No targets were set in advance — the first clean run *is* the baseline, so the gate
enforces "do not fall below this," not an invented number. The one exception is
grounding violations, which isn't a target but an invariant: it must be zero.

## Reading it

**What works:** decline rate is perfect — the reliability behaviour (declining rather
than inventing) holds. Mean candidates fell from 5 to 1.8; the list no longer pads.
Grounding is clean and now provably so.

**What doesn't:** noise rate 0.444 is high, but *concentrated, not spread* — two
specific descriptions (L2-003, L2-007) produce almost all the noise in the whole
suite. Every other entry is clean or nearly so. So the problem is two descriptions
the agent handles badly, not a diffuse quality issue — a located problem to work on
rather than a general sense that quality is mediocre.

## What this unblocks

Layer 2 now has a committed baseline across seven metrics and a clean grounding
invariant. The regression gate has something to regress against — and two concrete,
located problems to work on: the noise concentrated in L2-003 and L2-007, and the
XID-wraparound attractor (documented separately).
