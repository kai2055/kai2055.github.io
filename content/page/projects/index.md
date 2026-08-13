---
title: "Projects"
description: "Three layers of ML reliability — data, model, system."
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

ML systems fail at three points: bad data gets in, the model drifts in
production, or the same failure repeats because nobody learned from it. Each
project below hardens one of those points.

## 1. Data layer — Data Quality Checker

Catches bad input before it ever reaches a model.

*What breaks without it:* silent garbage-in-garbage-out — the model trains or
scores on corrupt data and no one notices until the numbers are wrong
downstream. Deployed on GCP Cloud Run.

→ [csv-health-tracker on GitHub](https://github.com/kai2055/csv-health-tracker)

## 2. Model layer — ML Reliability Pipeline

Keeps a model working after it ships, not just on the day it's deployed.

*What breaks without it:* a model that passed every test on launch day quietly
degrades in production, and the first sign is an angry user, not a metric.
Drift monitoring (PSI + Wasserstein), FastAPI serving, and CI/CD on GCP Cloud
Run. 110 tests, 26 ADRs.

→ [ml-reliability-pipeline on GitHub](https://github.com/kai2055/ml-reliability-pipeline)

## 3. System layer — Incident Post-Mortem Retrieval Assistant

Turns past failures into searchable, deployment-gating feedback.

*What breaks without it:* the same outage happens twice because the lesson from
the first one is buried in a wiki. A three-layer RAG system that retrieves,
diagnoses, and evaluates engineering post-mortems — built on LangGraph,
ChromaDB, and local LLMs via Ollama, with a RAGAS-backed evaluation framework.
*Layer 1 complete; diagnostic agent in progress.*

→ [incident-postmortem-assistant on GitHub](https://github.com/kai2055/incident-postmortem-assistant)

---

**How I build the foundation:** daily practice, retyped from spec, not skimmed —
[python-llm-guided-practice](https://github.com/kai2055/python-llm-guided-practice)
· [ml-study-lab](https://github.com/kai2055/ml-study-lab)
· [sql-practice](https://github.com/kai2055/sql-practice)
