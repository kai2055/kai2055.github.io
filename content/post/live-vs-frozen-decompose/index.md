---
title: "A Quality Gate Must Be Reproducible Before It Can Gate Anything"
description: "The regression runner's numbers wobbled run-to-run — not because the system changed, but because one step was non-deterministic. A gate that fires on its own noise gets muted."
slug: "live-vs-frozen-decompose"
date: 2026-07-26
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Finding
    - Evaluation
    - Reliability
---

> **The short version, for anyone:** This project has an automatic "quality gate"
> that blocks any code change which makes the system worse. But the gate's own
> measurements were bouncing around between runs — because one early step uses an AI
> model that phrases things slightly differently each time. So the gate couldn't tell
> a real problem from its own randomness, and on one run it "failed" for no real
> reason. The fix: freeze that non-deterministic step during measurement, so every
> run measures the same thing. The principle: *a gate whose own measurement isn't
> reproducible can't gate anything.*
>
> *Full finding below.*

---

**Surfaced by:** first full end-to-end run of the Layer 3 runner · **Related:**
ADR-016, ADR-020

## What happened

The first full run of the Layer 3 runner produced Layer 2 numbers that moved against
the committed baseline. The headline: **decline_rate dropped 1.000 → 0.500**, which
under ADR-020 is a hard invariant and would fail the gate. But it's only 2 no-match
entries scored, so 0.5 is literally one entry flipping — one no-match description
stopped declining and leaked a candidate.

Layer 1 reproduced almost exactly (hit rate 1.000, MRR 0.918 vs 0.9177). Layer 2 did
not.

## Why it moved — not a regression, a reproducibility gap

The movement is **not** a code or corpus regression. The store was freshly
re-indexed and Layer 1 reproduced perfectly, so retrieval is sound. The cause is that
**the runner ran the full live graph, including the Decompose node.**

Decompose is LLM-driven and non-deterministic. Its output — the symptom breakdown
that everything downstream retrieves against — varies run to run. So each run
measures against a *different* set of symptoms, and the metrics wobble accordingly.

The project already anticipated exactly this: a frozen symptoms file exists
specifically to hold Decompose's output fixed so measurement is reproducible. The
Layer 2 baseline was built against frozen symptoms. The runner was not — it bypassed
the freeze and called the live graph end to end. **So current and baseline were not
measuring the same thing.**

## Why this matters for the gate

The entire point of the gate is telling a real regression from noise. A gate that
runs live Decompose can never do that for Layer 2: every run drifts by an unknown
amount from Decompose alone, so any metric movement is ambiguous by construction.
decline_rate dropping below its hard invariant on this run is the proof — it *looks*
like a gate failure, but it's Decompose variance on a 2-entry denominator, not a
broken system. A gate that fires on its own measurement noise is the false-alarm
failure mode ADR-020 warns about. It would get muted.

## The fix

**The runner must measure Layer 2 against frozen symptoms, not live Decompose** — the
same way the baseline was built. Run the graph from the frozen symptoms starting
point rather than letting Decompose run fresh. This makes the runner reproducible and
makes current-vs-baseline an apples-to-apples comparison.

Remaining downstream LLM variance (Retrieve / Assess / Diagnose) is a deferred
question — whether it's small enough to gate on, or whether more of the chain needs
pinning, is answered once frozen-Decompose runs are compared across several repeats.
That repeat data is also what tightens the provisional thresholds in ADR-020.
