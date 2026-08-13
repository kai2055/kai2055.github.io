---
title: "Why Layer 2 Needed Its Own Threshold"
description: "The diagnostic agent was reusing a cutoff tuned for full questions — but it sends short symptom fragments, which score worse, so everything was being thrown away. And why 0.36 is a working setting, not a solution."
slug: "layer2-threshold"
date: 2026-07-24
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Finding
    - Retrieval
    - Evaluation
---

> **The short version, for anyone:** The diagnostic layer breaks an incident into
> short symptom fragments and searches for each. But it was reusing a
> "closeness" cutoff that had been tuned for full, richly-worded questions — and
> short fragments always score as less close, so **25 of 27 symptoms found the right
> incident and then had it thrown away**. Giving Layer 2 its own, looser cutoff fixed
> most of it. But the honest conclusion isn't "0.36 is the right number" — it's that
> *a closeness score alone can't cleanly separate signal from noise on short
> fragments*, because in the data the two genuinely overlap. Naming that limit is
> the point.
>
> *Full finding below.*

---

**Measured on:** 15 incident descriptions, 27 frozen symptoms, 5 no-match symptoms.

## The problem in one line

Layer 2 was using Layer 1's threshold. Layer 1's threshold was tuned on complete
questions. Layer 2 sends symptom fragments. Fragments score worse, so everything was
thrown away.

## Why fragments score worse

A post-mortem chunk describes a whole incident — trigger, failure, cascade, recovery
— many concepts in one paragraph. A complete question matches that richness. A
fragment matches one small part of it, so the distance is worse even though it
describes the same event. **The whole description is closer to the document than any
of its parts.**

| Input type | Typical distance |
|---|---|
| Layer 1 complete questions | 0.20 – 0.27 |
| Layer 2 symptom fragments | 0.32 – 0.41 |
| The threshold both were using | **0.30** |

The cutoff sits in the gap. Layer 1 clears it every time. Layer 2 never does.

## What was actually failing

| Verdict | Count | Meaning |
|---|---|---|
| PASS | **0** | found, kept, visible to the agent |
| THRESHOLD | **25** | found the right document, then discarded |
| RANK | 0 | found but ranked too deep |
| MISS | 2 | never found at all |

**25 of 27 symptoms found the correct document and had it thrown away.** Retrieval
was working. Filtering was calibrated for the wrong input — the correct document was
often ranked *first*, at distances like 0.32–0.33, just above the 0.30 cutoff.

## The sweep, and the decision

Each threshold was tested against the same frozen symptoms. The usable range came out
to **0.34–0.36**: below it, entries get no evidence; above 0.38 junk starts leaking
badly, and at 0.40 *every* no-match probe leaks — decline behaviour collapses
entirely. **0.36** is the best point: 9 of 13 entries get evidence, junk still only
1 of 5. Decision: `LAYER2_THRESHOLD = 0.36`, while Layer 1 keeps 0.30.

## What this does *not* fix

Stated plainly, because 0.36 is not a solution:

- **4 of 13 descriptions still get nothing** — the agent still fails on those.
- **4 symptoms return the wrong document** — the agent reasons over a wrong incident,
  which during an outage is worse than returning nothing.
- **1 junk symptom leaks** — a description with nothing matching gets treated as if it
  matched.

Layer 2 goes from never working to working roughly two-thirds of the time, with some
false leads.

## The overlap problem

The reason no threshold is clean: the distance bands *interleave*. A junk symptom
("air conditioning failed", 0.362) can score worse than a real one, but another junk
symptom ("machines shut themselves down", 0.334) scores *better* than a real hit
("servers dropping everything", 0.344). **No cutoff separates them, because the
separation doesn't exist in the data.** More samples would describe the overlap more
precisely; they wouldn't create a boundary.

So the honest conclusion is not "0.36 is the right number." It's: **a distance score
alone cannot separate signal from noise on short symptom fragments.** 0.36 is a round
number picked from where the table turns — a working setting that makes the component
functional, to be revisited when the suite grows. Confidence in the exact number is
low, deliberately so.

## What to try next

The threshold fix treats the symptom; the cause is that fragments carry less signal
than whole descriptions. The real direction: **search the full description alongside
each symptom** and merge the results — matching rich text against rich chunks, the
comparison the embedding model is actually good at. Evidence: the same content as
complete descriptions scored 0.870 at threshold 0.30, while split into fragments it
scores zero at the same threshold. That would widen the gap between signal and noise
rather than moving a line through the middle of it.
