---
title: "ML Reliability Pipeline"
description: "An early-warning system for a loan model judging today's borrowers by yesterday's world."
slug: "ml-reliability-pipeline"
date: 2026-02-12
layout: "page"
---

**What breaks without it →** A bank trains a loan-approval model on years of
history, ships it, and it passes every test. Then the world moves — a pandemic, a
rate shock, a recession — and the model doesn't notice. It keeps approving and
rejecting 2020 applicants using patterns it learned from 2019, as if nothing
changed. It still looks healthy on a dashboard. The first real sign something's
wrong is a wave of defaults nobody saw coming.

**What this delivers →** A system that continuously checks whether today's loan
applicants still look like the ones the model was trained on — and the moment they
stop matching, it says so, naming exactly which factors have shifted. A stale
model becomes an *alert on a screen* instead of a *loss on the books*.

## The problem, in plain terms

Most ML portfolios show a model that works once, on a clean test set. The harder,
more valuable question is the one a lender actually faces: *the model worked in
2019 — how would you even know it stopped working in 2020?* By the time bad loans
turn into defaults, the damage is done. You need to catch the drift **before** it
reaches the loan book, not after.

## What I built

An end-to-end loan-default pipeline that trains on U.S. Small Business
Administration 7(a) loan data from 2010–2019, then watches 2020-onward
applications for drift against that baseline. Four layers, each with one job:

- **Data** — loads and validates every extract against a frozen schema, so if the
  government changes the file format, the pipeline fails loudly instead of quietly
  training on the wrong thing.
- **Models** — trains, tunes, evaluates, and selects a model, with every run
  tracked in MLflow and saved as a versioned artifact.
- **Monitoring** — snapshots the 2010–2019 baseline at training time, then scores
  incoming loans against it and applies a drift policy to decide what counts as a
  real, significant shift versus normal noise.
- **Serving** — a FastAPI app exposes prediction and drift endpoints, deployed
  live on GCP Cloud Run.

## What the monitor actually caught

This is the payoff, and it's a real story, not a hypothetical. When the pipeline
scored post-2020 loan data against its pre-pandemic baseline:

- **7 of 12 features had drifted significantly** (PSI > 0.25, a standard
  statistical threshold). In plain terms: more than half of what the model leaned
  on to judge a borrower no longer meant what it used to.
- The loudest signal was the **initial interest rate**, which had **shifted 1.46
  standard deviations** from its pre-COVID normal — the lending environment itself
  had changed, and the model had no idea.
- A model frozen in 2019 was handing down 2020 verdicts on outdated assumptions —
  and instead of finding that out from a default report a year later, the monitor
  named precisely which assumptions had broken, and by how much.

That's the concrete value: the difference between *"our defaults spiked last
quarter, why?"* and *"seven of our model's inputs drifted after COVID — here's the
list, flagged in real time."*

## A note on honesty

The baseline model scores a ROC-AUC of 0.9721. That looks spectacular — which is
exactly why I flagged it as a **red flag, not a trophy**. On loan-default
prediction, accuracy that high almost always means *label leakage*: some feature
quietly encoding the answer the model is supposed to predict. The leakage audit is
scoped for v2; until then the model is deliberately treated as scaffolding —
something for the monitoring layer to watch. *Reliability engineering is partly
about not being fooled by your own good-looking numbers, so this caveat stays in
on purpose.*

## Key decisions (the "why," not just the "what")

Every non-trivial choice is written down as an Architecture Decision Record —
**27 of them**. A few that show the range:

- **ADR-012** — the schema is the single source of truth; everything validates
  against it.
- **ADR-021** — how the model and its approve/reject threshold get selected.
- **ADR-027** — why v1 deliberately stops at *detecting* drift instead of
  automatically retraining. Retraining on drifted, possibly-leaky pandemic data
  could make the model *worse* — so the system flags a human instead of silently
  "fixing" itself.

Code tells you *what*; six months later, the ADRs are the only thing that tells
you *why*.

## Engineering quality

- **112 tests**, including slow integration tests that run both pipelines
  end-to-end.
- **CI on every push**: lint (ruff), type-checking (mypy), tests with a 75%
  coverage gate, spell-check, and a Docker build check.
- Immutable config objects, fixed seeds, explicit statistical conventions, UTC
  timestamps — reproducibility by default.

## What this demonstrates

Not just building a model, but building the *system that keeps a model trustworthy
once real money rides on it* — data validation, drift monitoring, versioned
experiments, a live API, and a documented decision trail. This is the
day-two-onward work that MLOps and ML reliability roles actually hire for.

→ **[Live API](https://ml-reliability-pipeline-1061232555311.europe-west1.run.app/docs)** · **[GitHub repo](https://github.com/kai2055/ml-reliability-pipeline)**
