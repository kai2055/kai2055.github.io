---
title: "Projects"
description: "Four projects, one theme — building ML systems that stay working, not just work once."
slug: "projects"
date: 2026-02-01
layout: "page"
menu:
    main:
        weight: -80
        params:
            icon: code
---

**I build ML systems that don't just work — they stay working.**

ML systems fail at predictable points: bad data gets in, a model quietly drifts
out of date, past failures get forgotten, or a live data feed shifts under the
model's feet. Each project below hardens one of those points — and each has a
full write-up, in plain language up top and full engineering depth below.

---

### [ML Reliability Pipeline](/ml-reliability-pipeline/)
**An early-warning system for a loan model judging today's borrowers by yesterday's world.**

Picture a bank approving small-business loans with a model trained on 2010–2019
data. Then COVID hits — and the model keeps scoring 2020 applicants as if it were
still 2019. On real U.S. loan data, this system caught the shift directly:
**7 of 12 factors the model relied on had drifted significantly**, the biggest by
**1.46 standard deviations**. A stale model becomes an alert on a screen instead
of a loss on the books.

*Python · PSI drift detection · MLflow · FastAPI · Docker · GCP Cloud Run · 112 tests · 27 ADRs*
→ **[Read the case study](/ml-reliability-pipeline/)** · **[GitHub](https://github.com/kai2055/ml-reliability-pipeline)**

---

### [Incident Post-Mortem Retrieval Assistant](/incident-postmortem-assistant/)
**When a system crashes at 3 a.m., it answers "have we seen this before?"**

Every outage gets written up — then buried in a wiki nobody can search during the
*next* one. This tool lets an engineer describe a failure in plain English and
returns the most relevant past incidents, their root causes and fixes — and
crucially, refuses to answer when it has no real match, instead of inventing a
convincing wrong one. Retrieval hits the right incident **every time (hit rate
1.00)**, and answer quality is enforced on every code change.

*Python · RAG · LangGraph agent · ChromaDB · local LLMs (Ollama) · 39-query eval suite · quality-gated CI*
→ **[Read the case study](/incident-postmortem-assistant/)** · **[GitHub](https://github.com/kai2055/incident-postmortem-assistant)**

---

### [CSV Health Tracker](/csv-health-tracker/)
**Stops broken data at the door, before it poisons everything downstream.**

When a data file shows up half-empty or full of duplicates, a model doesn't
complain — it trains on the garbage and produces confident, wrong results. This
tool checks any incoming CSV *before* it reaches the pipeline. On a real
545,000-row government dataset, it instantly flagged five columns that were
**76–97% empty** — the kind of silent rot that wrecks a model trained on it.

*Python · FastAPI · Docker · GCP Cloud Run · configurable YAML thresholds · 16 tests*
→ **[Read the case study](/csv-health-tracker/)** · **[GitHub](https://github.com/kai2055/csv-health-tracker)**

---

### [Berlin Transit Delay Intelligence](/berlin-transit/) · *In progress*
**Predicting how one delayed train becomes ten — before it happens.**

In a transit network, a single delay never stays single: one held S-Bahn ripples
into missed connections across the city, and operators usually see the cascade
only as it unfolds. This project (my current build) predicts how a delay
propagates through Berlin's network and flags abnormal disruptions as they
emerge — the reliability thread applied to live, constantly-shifting data, the
hardest kind to keep a model honest on.

*Airflow · PySpark · AWS S3 · PostgreSQL · dbt · XGBoost · MLflow · Kubernetes · Prometheus + Grafana*
→ **[See the roadmap](/berlin-transit/)** · **[GitHub](https://github.com/kai2055/berlin-transit)**
