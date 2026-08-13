---
title: "The Test Suite That Was Too Slow To Run"
description: "Every test passed and every test was correct — and the suite took 15m 46s, which in practice means nobody runs it. Getting it to 8.5s."
slug: "test-suite-too-slow"
date: 2026-07-24
categories:
    - Deep Dives
tags:
    - Incident Assistant
    - Case Study
    - Testing
    - Reliability
---

> **The short version, for anyone:** A project's automated tests are its safety
> net — but only if they get run. This suite took over 15 minutes, which in
> practice means developers skip it, which means the safety net isn't there. Every
> test was correct; the problem was speed. This is the story of cutting it from
> **15m 46s to 8.5 seconds** — not by removing tests, but by noticing the same
> expensive work was being redone dozens of times. The theme: *correct is not the
> same as usable.*
>
> *Full technical walkthrough below.*

---

**Result:** Fast suite 15m 46s → 8.5s · **Also found:** an 82-vs-83 chunk
discrepancy open since early July.

## Framing: this was not a bug

Every test passed. Every test was correct. Nothing produced a wrong answer. The
defect was that the suite took **15 minutes and 46 seconds**, which in practice
means you stop running it. A test suite you avoid running provides no safety at all,
however correct it is.

So this is a performance defect in the test harness, not a bug in the system —
worth being precise about, because the fix is different in kind. Nothing was
repaired. Work that was being done repeatedly was made to happen once.

## The symptom

```
85 passed, 1 deselected in 946.89s (0:15:46)
```

The project's stated test architecture is pure-core / impure-shell: pure logic
tested in milliseconds, slow integration tests run deliberately. The 29 agent tests
already demonstrated this — fully mocked, 0.19 seconds. So the architecture was
right and the suite still took a quarter of an hour. Something was not matching the
design.

## Measuring instead of guessing

```bash
pytest tests/ -m "not integration" --durations=20
```

`--durations` prints the slowest tests. The output split cleanly into two groups,
and the distinction between them turned out to be the whole story.

**Group 1 — time in `setup`:** four tests, ~90–110 seconds each, all in setup.

**Group 2 — time in `call`:** four tests, ~105–217 seconds each, in the test body.

And once setup was done, the tests themselves ran in **under two seconds**. The
tests were never slow. The setup was.

## Cause 1: a fixture doing the same work eight times

```python
@pytest.fixture
def indexed_chunks(temp_chroma, corpus_path, expected_chunks):
    docs = load_documents(corpus_path)
    chunks = chunk_documents(docs)
    index_chunks(chunks)
    return chunks
```

No `scope` argument, so pytest defaults to function scope: the fixture runs fresh
for every test that requests it. Embedding 82 chunks through nomic-embed-text on CPU
costs roughly 100 seconds. Four tests used the fixture. Four identical embedding
passes, ~400 seconds, for one corpus that never changed between them.

**Fix — session scope**, so the corpus is embedded once and shared. One
complication: `temp_chroma` used pytest's function-scoped `monkeypatch`, which a
session-scoped fixture can't consume, so the replacement instantiates
`pytest.MonkeyPatch()` directly and undoes it manually.

**The tradeoff, stated plainly:** function scope exists for isolation — every test
gets a clean store. Session scope trades that away for speed. Acceptable here
because these tests read from the store rather than corrupting it — and because a
suite nobody runs is worse than a small isolation risk. Result: 15m 46s → 11m 25s.

## Cause 2: a test embedding 82 chunks to count to 82

```python
def test_vector_count_matches_chunk_count(corpus_path, expected_chunks):
    docs = load_documents(corpus_path)
    chunks = chunk_documents(docs)
    chunks_with_vectors = embed_chunks(chunks)
    assert len(chunks_with_vectors) == expected_chunks
```

110 seconds. Read the assertion: it checks that `embed_chunks` returns one vector
per chunk it was given. It never inspects a single vector. That property does not
depend on corpus size — five chunks prove it as well as 82. Slicing to `chunks[:5]`
took it from **110s → 4.67s**, and the assertion got *better*: the real property is
"same number out as in," not "82."

## Cause 3: genuinely slow tests were not labelled

Three tests could not be shrunk — `test_store_and_search`, `test_search_with_filter`,
`test_idempotency` — because they genuinely need a fully populated store; that's the
thing under test. They needed a *label*, not an optimisation:

```ini
[pytest]
markers =
    slow: embeds the corpus, needs Ollama
    integration: full end-to-end run across components
```

Three speeds instead of two:

```bash
pytest -m "not slow and not integration"   # 8.5s   - run constantly
pytest -m "slow"                            # ~5 min - before committing
pytest -m "integration"                     # ~100 min - deliberately
```

The middle tier is the one that was missing. Previously "not integration" meant 15
minutes, so the only real choice was 15 minutes or nothing.

## Result

| Stage | Time |
|---|---|
| Start | 15m 46s |
| After session-scoped fixtures | 11m 25s |
| After rewriting the count test | ~9m |
| After marking slow tests | **8.5s** |

78 tests run by default. 8 parked behind markers.

## The side finding: 82 vs 83

`conftest.py` expects 82 chunks and the chunking test passes — so chunking really
does produce 82. But the live store at `data/chromadb` held **83**. This
discrepancy had been open since early July. It's now explained: the corpus is
correct, and the live store contained one orphan chunk from an earlier ingest — a
document later changed or removed without the store being rebuilt. A clean re-index
resolves it, worth doing before further calibration, since a stale chunk can surface
in retrieval and quietly distort a measurement.

## What this is a story about

**Correct is not the same as usable.** Every test passed and asserted something
true. The suite still failed at its actual job — being run often enough to catch
regressions early. **Measure before optimising:** the `setup`-vs-`call` split
pointed straight at two different causes needing two different fixes. **Ask what a
test is actually asserting:** the 110-second count test verified a property that had
nothing to do with corpus size. **And some slowness is real** — three tests genuinely
need a populated store, so the fix was a marker, letting them run deliberately
rather than be skipped by accident.
