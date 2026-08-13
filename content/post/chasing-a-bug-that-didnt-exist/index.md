---
title: "Chasing a Bug That Didn't Exist"
description: "The retrieval code was correct — and still useless. The real culprits were an eval suite grading itself too kindly and a silent truncation dropping right answers."
slug: "chasing-a-bug-that-didnt-exist"
date: 2026-07-23
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Bug Report
    - Evaluation
    - Retrieval
---

> **The short version, for anyone:** A search system was returning "nothing
> found" — while the correct answer sat in its database, ranked first. Nothing had
> crashed; every function did exactly what it was written to do. This is the story
> of discovering that *the code was fine and the system was still broken* — because
> the test suite had been unknowingly grading the system on easy questions, and a
> hidden limit was throwing away correct answers before anyone looked at them. It's
> a good example of the kind of failure that doesn't show up as an error message —
> the most dangerous kind.
>
> *Below is the full technical investigation. Skip it freely — the summary above is
> the point.*

---

**Layer:** 1 (retrieval) · **Outcome:** No defect found in application code. Two
real problems found anyway.

## Where it started

Layer 2 was blocked. A retrieval probe returned zero results against a store
holding 83 chunks. No error, no exception — just an empty list.

The working theory was environmental: the embedding model wasn't loaded in Ollama.
That theory mattered, because Layer 2's "the agent declines honestly" framing
depended on knowing whether the empty result was a genuine no-match or a broken
pipe.

First command of the day killed it. `ollama list` showed `nomic-embed-text`,
274 MB, installed four weeks earlier. The model was fine. So the bug was real, and
somewhere in code I'd written.

## The elimination

Rather than guess, I listed every link in the chain and tested each in order. Six
suspects.

| Suspect | Result |
|---|---|
| Model not installed | Present |
| Wrong database path | App points at `data/chromadb` — correct |
| Store empty | 83 chunks |
| Wrong distance measure | `hnsw:space: cosine`, set explicitly |
| Embedding prefix mismatch | `search_document:` / `search_query:` correctly paired |
| Threshold comparison inverted | `distance <= threshold` — right direction |

Every single one came back clean. Every function I checked did exactly what it was
written to do. That was the first real finding: **the system was correct and still
useless.**

## Three false leads, all self-inflicted

The diagnostic script lied to me three times, and each time for the same reason.

**It hardcoded the database path.** Pointed at `data/chroma` instead of
`data/chromadb`. ChromaDB doesn't error on an empty folder — it silently creates a
blank database. So the script made an empty store and correctly reported it was
empty. Cost: half an hour convinced the corpus had vanished.

**It reimplemented the embedding call.** Called Ollama directly with plain text,
no prefix. The corpus was embedded with `search_document:`. So it was comparing
unlabelled queries against labelled documents — a mismatched measurement that
produced a real-looking number I then reasoned from.

**It used a candidate depth the system doesn't use.** I set 15 to see where
documents ranked. The system uses 5. That made the diagnostic disagree with the
sweep, and reconciling the two is what exposed the actual bug.

The lesson generalises: **a diagnostic that restates what the system defines will
eventually disagree with it, and it will disagree quietly.** Every fix was the same
— import the real code instead of copying it.

## What was actually wrong

Two things, neither a broken function.

### The evaluation suite was easier than reality

The 31-query suite was written after reading and normalising all 15 post-mortems.
So it reuses the documents' own vocabulary without meaning to. It scored the system
1.000 while plain-English questions were failing.

To measure this instead of asserting it, I built a paired suite: same entries, same
`expected_doc_id`, same difficulty, only the query text rewritten in plain English.
No-match probes and filter queries held constant as a control.

| Suite | Hit rate @ 0.30 | MRR |
|---|---|---|
| Original wording | 1.000 | 0.949 |
| Plain English | 0.826 | 0.783 |

The control held — decline rate identical across both suites at every threshold. So
the gap came from wording and nothing else.

The clearest single illustration:

| Query | Distance |
|---|---|
| "BGP route leak" — the document's own words | 0.1996 |
| "database ran out of connections" — same kind of event, plain words | 0.3035 |

The distance was largely measuring word overlap. The threshold was acting as a
jargon filter.

### `top_k` was discarding correct answers before checking them

The plain-English curve flattened at 0.913 and never moved, even at threshold 0.50
where the filter does nothing. That meant some failures weren't threshold failures
at all.

A per-query diagnostic found the reason: one query's correct document sat at **rank
6, distance 0.2888** — comfortably inside the 0.30 threshold, never looked at,
because `top_k=5` truncated the list first.

The candidate count was overriding the relevance rule. Raising it to 10:

| | Before | After |
|---|---|---|
| Hit rate @ 0.30 | 0.826 | **0.870** |
| Ceiling | 0.913 | **0.957** |
| Decline rate | 0.600 | 0.600 |

Real answers gained, no noise admitted. The same number turned out to be hardcoded
in four places — `retrieve()`, `run_sweep`, `score_filter_query`, and the agent's
retrieve node at 3. It's now one constant, `DEFAULT_TOP_K`.

Worth noting the agent was retrieving **three** candidates per symptom while Layer 1
used five. The component doing the harder work had the least evidence.

## The decision I didn't make

Two queries still fail at 0.30, missing by 0.0055 and 0.0092. Moving the threshold
to 0.35 would recover both. I left it alone.

The distance bands overlap — a junk probe scored 0.215, closer than four of five
real queries. There is no value that separates good from bad, so any number is a
tradeoff, and one tuned to clear 23 specific queries is fitting the sample rather
than the problem. 0.35 would also drop decline rate from 0.600 to 0.200 — triple the
noise to recover two borderline cases.

And the failure modes aren't equal. Returning nothing costs an engineer time.
Returning the wrong past incident during a live outage sends them after the wrong
root cause. Strict is the right side to fail on.

## What this is actually a story about

Not a bug hunt. Every function was correct. It's about a system that stays up,
throws no errors, and quietly returns nothing while holding the answer. During a
live outage it would tell an engineer that nothing similar has ever happened — with
the matching post-mortem in hand, ranked first.

And it's about the evaluation framework marking its own homework. The suite said
1.000. The system was at 0.826 for anyone who hadn't read the corpus. The
measurement was wrong in the flattering direction, which is the direction you don't
check.

The fix for that isn't a better threshold. It's a regression gate — re-running these
measurements on every corpus change and reporting when the numbers move. You don't
calibrate once; you build the thing that notices when calibration has gone stale.
