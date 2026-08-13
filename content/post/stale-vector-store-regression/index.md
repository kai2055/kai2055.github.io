---
title: "A Stale Vector Store Caused a Phantom Regression"
description: "Passing tests suddenly failed right after a code refactor. The refactor was innocent — a gitignored database had silently gone out of date."
slug: "stale-vector-store-regression"
date: 2026-07-26
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Finding
    - Evaluation
    - Retrieval
---

> **The short version, for anyone:** Tests that had been passing suddenly started
> failing, right after a code change — so the code change looked guilty. It wasn't.
> The real cause was a database of pre-computed data that had quietly gone stale: it
> was excluded from version control, so the usual "has anything changed?" check
> showed everything clean while the thing the system actually depends on was out of
> date. The takeaway: *a clean version-control status tells you nothing about the
> state of files it doesn't track* — and this is exactly the failure the project's
> automatic quality gate is built to prevent.
>
> *Full finding below.*

---

**Surfaced by:** first Layer 2 runs while building the Layer 3 runner · **Related:**
ADR-020 (Layer 3 gate policy)

## What happened

Mid-session, Layer 2 entries that pass in the committed baseline started failing.
Two examples: **L2-006** went from a correct top-1 hit to **0 candidates, declined,
retrieved nothing**; **L2-001** went from the correct incident in its candidate set
to 3 candidates, all wrong.

On the surface this looked like a regression — the system had gotten worse — and it
appeared right after a code refactor, so the refactor was the obvious suspect.

## Why it was NOT the obvious cause

The refactor was innocent. It only moved a loop (`run_suite` extracted from `main`);
it touched no retrieval or graph code. Isolation confirmed this:

- `ollama list` — the embedder was loaded and fine.
- `git status` / `git log` — corpus untouched since the baseline commit. No data
  change.
- A *second* entry also mis-retrieved, so it wasn't one bad entry — it was systemic
  to retrieval.
- Comparing to the committed baseline showed the affected entries **used to pass**.
  So the baseline was right and *current* had drifted — pointing away from a bad
  baseline and toward something environmental.

## Root cause

The **vector store on disk was stale.** The corpus had grown from 15 to 20 documents
in an earlier commit, but the ChromaDB index at `data/chromadb` had never been
rebuilt after that growth. So retrieval was searching the **old 15-doc index** while
the suite and baseline expected the **current 20-doc / 107-chunk** corpus. Wrong
incidents came back, or none at all.

The reason `git status` gave no warning: **`data/chromadb` is gitignored** — it's a
generated artifact, not tracked. So git reported a clean tree while the actual thing
retrieval depends on was silently out of date. **A clean git tree says nothing about
the state of the index.**

## The fix

Re-index so the store matches the current corpus (`python -m src.embedding`). After
re-indexing, L2-001 immediately found the expected incident again, and the full suite
reproduced documented behavior.

## Why this is load-bearing for the quality gate

This is exactly the failure the Layer 3 runner's **mandatory re-index step** prevents.
A gate that evaluated without re-indexing would compare a fresh baseline against a
possibly-stale store and report regressions that aren't real — the false-alarm
failure mode ADR-020 warns about. The re-index isn't hygiene; it's a *correctness
precondition*, now demonstrated rather than assumed.

## Lessons

- **Gitignored artifacts have no version signal.** A clean tree can sit on top of a
  stale generated dependency. Never infer store freshness from git.
- **Diagnose environmental vs. logic failures before concluding.** The symptom framed
  the refactor as guilty; the cause was a stale artifact. Isolation, not assumption,
  found it.
- **The baseline was the tool that cracked it.** Run current, diff against committed
  baseline, find the flipped entry, isolate the cause — the same procedure the quality
  gate automates, run here by hand.
