---
title: "Confident Matches on Vocabulary, Not Mechanism"
description: "The system sometimes matches on shared words rather than shared cause — with high confidence. A known limitation kept visible on purpose, because it's the exact failure the project exists to guard against."
slug: "vocabulary-not-mechanism"
date: 2026-07-25
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Finding
    - Retrieval
    - RAG
---

> **The short version, for anyone:** The system finds past incidents by comparing
> *meaning*, but sometimes two very different failures use the same words —
> "Cloudflare," "edge," "database" — and it confidently returns the wrong one. This
> is the most dangerous kind of error: not a crash, not an obvious miss, but a
> confident wrong answer that would send an engineer down the wrong path during a
> live outage. Rather than hide it or fake a better score by deleting the test, I've
> kept it visible and documented — because knowing exactly where a reliability
> system fails *is* the reliability work, and this particular fix is real design
> work, not a quick tuning tweak.
>
> *Full finding below.*

---

**Where it shows:** Layer 1 retrieval and Layer 2 diagnosis — the same failure,
twice. **Status:** Understood, not fixed. Kept visible on purpose.

## The short version

The system sometimes matches on shared words rather than shared cause, and does it
with high confidence. A query and an incident can use the same vocabulary while
describing completely different failures. The embedding scores them as close, the
system returns the wrong incident, and nothing about the score signals that it's
wrong. This is the failure the whole project exists to guard against: not a crash,
not an obvious miss, but a confident wrong answer that would mislead an on-call
engineer during a live outage.

## Instance 1 — Layer 1 retrieval: the DDoS probe

A no-match probe: *"DDoS attack overwhelmed our CDN edge nodes and caused a 12-hour
outage."* This is meant to retrieve nothing — the corpus has no DDoS incident.
Against the 15-document corpus it correctly declined. Against the 20-document corpus
it now matches a Cloudflare incident at distance **0.236** — well inside the 0.30
threshold, a confident hit.

But that Cloudflare incident is a **configuration-error** incident: a database
access-control change that cascaded. It has nothing to do with a DDoS, which is a
volumetric attack. The match is on surface vocabulary — "Cloudflare," "edge,"
"outage" — not on the failure mechanism. The probe stays classified as a no-match;
its continued matching is the finding, not something to reclassify away.

## Instance 2 — Layer 2 diagnosis: the XID-wraparound attractor

In the Layer 2 baseline, three descriptions produced a diagnosis of Postgres
transaction-ID (XID) wraparound. Two were correct (a real Sentry Postgres incident).
The third, L2-007, was **wrong** — it's Roblox's service-registry cascade, nothing to
do with Postgres. But it shares symptom vocabulary — read-only, cascade, database —
with the Sentry incident, and the model reached for the specific,
authoritative-sounding failure it had seen before.

## Why these are the same failure

Both are the system latching onto a **specific, plausible, well-documented failure**
because the surface features match, while the actual mechanism does not. The danger
in both is the **confidence**. A vague wrong answer is easy to distrust. "PostgreSQL
XID wraparound" and a 0.236 distance both look authoritative. During an incident,
that's worse than silence — it's a false lead delivered with conviction.

## Why it is being kept visible rather than patched

**It is honest signal.** The Layer 1 decline rate is 0.500 — three of four no-match
probes decline correctly, and the DDoS probe is the one that doesn't. Forcing that
number to look better by deleting the probe would hide a real property of the system.
The metric is more useful with the known failure inside it.

**The fix is not local.** This isn't a threshold to nudge or a prompt line to add.
It's a limitation of matching on embedding similarity and symptom vocabulary.
Addressing it properly means giving the system more to discriminate on — richer
context at retrieval time, or a verification step that checks whether the mechanism
actually fits before returning a confident answer. That's real design work, recorded
here as the direction rather than attempted as a patch.

## What would actually address it

- **Retrieval:** search richer text so the match rests on more than a few shared
  nouns. More context gives the embedding more to separate genuinely-similar
  incidents from merely-similarly-worded ones.
- **Diagnosis:** a mechanism-check before a candidate is returned with high
  confidence — does the cited incident's actual failure mode match the symptoms, or
  only their vocabulary? This is a grounding step one level deeper than
  citation-checking: not "is this incident real and retrieved," but "does this
  incident actually *explain* what was reported."

Both are deferred. Both are the honest fix. Neither is a tuning change.
