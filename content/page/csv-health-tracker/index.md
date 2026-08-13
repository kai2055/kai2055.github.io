---
title: "CSV Health Tracker"
description: "Stops broken data at the door, before it poisons everything downstream."
slug: "csv-health-tracker"
date: 2026-02-10
layout: "page"
---

**What breaks without it →** A data file lands in the pipeline with a quarter of
its values missing. Nothing errors out — the model just quietly trains on
incomplete data and starts making worse decisions. A week later someone notices
the numbers are off, and the hunt begins: by the time anyone traces it back to
that one bad file, hours are gone. Bad data is cheap to catch at the door and
expensive to debug once it's inside.

**What this delivers →** A fast, automatic quality gate that inspects a CSV
*before* it enters the pipeline and reports exactly what's wrong — which columns
are empty, how many rows are duplicated, where the whitespace pollution is — so a
bad file gets caught and rejected up front, not discovered downstream after it's
already done damage.

## What it does

Runs a configurable validation pass over any CSV and flags the common, silent
data-quality problems:

- **Missing values** — per column and overall
- **Duplicate rows**
- **Whitespace pollution** — stray leading/trailing spaces that break joins and
  lookups
- **File accessibility** — path, extension, and readability

It never touches the source data — it only reports on it. And every strictness
threshold lives in a config file, so tightening or loosening the checks needs *no
code change and no redeployment*.

## What a real run looks like

Pointed at the full U.S. Small Business Administration 7(a) loan dataset —
**545,751 rows, 43 columns** — it passed the duplicate check (0.21%, well under
the limit) but immediately flagged the rot:

- `bankncuanumber` — **96.9% empty**
- `chargeoffdate` — **93.2% empty**
- `franchisename` — **91.3% empty**

Those are columns a model would otherwise treat as real signal. Catching them
takes seconds here; missing them costs days later.

## Why it's deliberately small

This was the **first project in a focused MLOps sprint**, and the modest scope was
the point. It started as a single throwaway script and was rebuilt into a modular,
containerised, tested service deployed on the cloud with CI/CD — the goal being to
*practice production patterns on a problem I could fully control*, rather than to
wrestle a hard problem with sloppy engineering.

That intent shows in one deliberate choice: **both versions are kept side by side
in the repo** — `v1-simple-script` (the throwaway prototype) next to `v2-modular`
(the production refactor, with validation logic, reporting, configuration, and
error handling each cleanly separated). The diff between them shows the
architectural reasoning more clearly than any written explanation could.

## Engineering quality

- **16 tests**, each module with its own suite. The validation tests matter most:
  every check has a fixture that deliberately triggers it, so a broken check *fails
  loudly* instead of silently waving bad data through.
- **Custom exception hierarchy** — failures surface as distinct, actionable
  messages, not a generic stack trace.
- **Deployed live** on GCP Cloud Run, so a file can be checked via a hosted API
  without cloning anything.

## What this demonstrates

Turning a ten-line script into a properly engineered, tested, containerised,
deployed service — the foundational discipline the rest of my project sprint
builds on. Small problem, production-grade execution.

→ **[Live API](https://csv-health-tracker-127482995435.europe-west3.run.app/docs)** · **[GitHub repo](https://github.com/kai2055/csv-health-tracker)**
