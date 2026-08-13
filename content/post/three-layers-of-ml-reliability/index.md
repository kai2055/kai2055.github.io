---
title: "The three layers where ML systems fail"
description: "Bad data, silent drift, and lessons that never stick — and how to harden each."
date: 2026-02-12
categories:
    - MLOps
tags:
    - MLOps
    - Reliability
    - Drift Detection
    - RAG
---

Most ML portfolios show a model that runs once. The interesting engineering is
in the part that comes after: keeping it running. In my experience, ML systems
fail at three predictable points.

## 1. Bad data gets in

The model trains or scores on corrupt input and nobody notices until the numbers
are wrong three steps downstream. The fix is boring and essential: validate data
*before* it reaches the model — missing values, duplicates, malformed headers,
schema drift.

## 2. The model drifts in production

A model that passed every test on launch day quietly degrades as the world
changes underneath it. The first sign shouldn't be an angry user — it should be
a metric. That means monitoring distribution shift (PSI, Wasserstein) and gating
deployments on it.

## 3. The same failure repeats

An outage happens, someone writes a post-mortem, it gets buried in a wiki, and
six months later the same thing happens again. Turning past failures into
*searchable, deployment-gating* feedback is the third layer — and the one almost
nobody builds.

---

One principle — reliability — across three layers: data, model, system. That's
the thread running through everything on my [projects](/projects/) page.
