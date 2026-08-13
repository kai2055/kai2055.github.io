---
title: "Berlin Transit Delay Intelligence"
description: "Predicting how one delayed train becomes ten — before it happens. (In active development.)"
slug: "berlin-transit"
date: 2026-02-16
layout: "page"
---

> **Status: in active development.** This page describes the problem and the
> architecture being built. Measured results will be added as the pipeline
> produces them — I don't publish numbers I haven't measured.

**What breaks without it →** A transit operator (or a mobility app routing
thousands of riders) treats delays as they happen: a train is late, a platform is
crowded, a line is disrupted — each handled in isolation, in the moment. But delays
*cascade*, and the cascade is predictable in ways a human watching live boards
can't track. So the response is always reactive: by the time the knock-on delays
are visible, the window to reroute riders or hold a connecting service has already
closed.

**What this delivers →** A system that predicts how a delay will spread across the
network *before* it fully propagates, and detects anomalies — disruptions that
don't look like normal daily variation — as they emerge. The operator gets a head
start; the rider gets rerouted before they're stranded.

## The problem, in plain terms

Public-transit data is *non-stationary* — it never stops changing. Rush hour
differs from midnight, Monday from Sunday, summer from a snowed-in January, a
normal day from a strike or a stadium event. That's exactly what makes it a hard,
honest reliability problem: a model trained on last month's patterns is
*constantly* at risk of drifting out of date, because the world it's predicting
genuinely shifts under it every day. It's the natural sequel to my drift-detection
work — but on live data instead of a frozen historical dataset.

## What I'm building

A full production-style data platform, deliberately built on the tools these
systems use in industry:

- **Ingestion & orchestration** — Airflow schedules regular pulls of
  Berlin/Brandenburg (VBB/DB) transit data into a raw landing zone on AWS S3.
- **Processing** — PySpark handles the heavier transformations; dbt shapes the
  cleaned, modelled tables in a PostgreSQL warehouse.
- **Modelling** — XGBoost for delay-propagation prediction, with statistical
  anomaly detection for unusual disruptions; experiments tracked in MLflow.
- **Serving & monitoring** — a FastAPI service and a Streamlit interface, with
  Prometheus + Grafana for live observability, deployed on Kubernetes.
- **The reliability throughline** — drift and anomaly detection running on
  genuinely live, non-stationary data: the discipline from my earlier projects,
  applied where it's hardest.

## Status & roadmap

*(Honest, current state — updated as it progresses.)*

- [ ] Ingestion pipeline (Airflow → S3)
- [ ] Warehouse + dbt models (PostgreSQL)
- [ ] Delay-propagation model (XGBoost) + baseline metrics
- [ ] Anomaly detection + drift monitoring
- [ ] Serving, dashboards, and deployment (FastAPI / Streamlit / Grafana /
  Kubernetes)

Once the pipeline produces measured results, this section is replaced by the same
evidence-first treatment as my other case studies — real numbers, the trade-offs
behind them, and the decisions that got there.

## What this will demonstrate

End-to-end data engineering on the modern industry stack (Airflow, PySpark, dbt,
Kubernetes, cloud storage) *combined* with the ML-reliability discipline running
through all my work — on the hardest data to stay reliable on: live and always
changing.

→ **[GitHub repo](https://github.com/kai2055/berlin-transit)**
