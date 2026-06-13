---
date:
tags: pandas memory-leak long-running-process gc.collect dataframe
one_liner: A pandas DataFrame held alive across a 24h cache rebuild cycle pushed RSS from ~62 MB to 1.2 GB on a constrained VPS.
---

# Pandas DataFrame not freed between cycles — memory grows to 1.2 GB on a 3.7 GB VPS

**Severity:** HIGH
**System layer:** agent loop — predictive score module

---

## Symptom

The agent process RSS grew steadily over time, eventually reaching a peak of 1 222 MB on a VPS with 3.7 GB total RAM. The growth was not constant — it followed a sawtooth pattern tied to a 24h cache rebuild cycle. Memory climbed during the rebuild, then did not fully return to baseline. After several days, the process was visibly consuming a disproportionate share of available memory.

No OOM kill occurred, but the risk was real. No alert fired automatically; the issue was found during a manual memory audit.

---

## Diagnosis

The agent maintains a predictive score module that caches historical probability data. The cache has a 24h TTL. When it expires, the module fetches several thousand OHLCV candles, builds a probability matrix from the raw DataFrame, then writes the result to a JSON cache file.

The problem: the DataFrame was passed into `build_probability_base()`, the scalar results were extracted — but the original `df` variable was never explicitly released. Python's reference counting kept the object alive for the lifetime of the function scope, and CPython's cyclic GC did not reclaim it promptly in a long-running process context.

A secondary source: the exchange client was re-instantiated on each market data call (~58 times per day), each instantiation carrying a memory footprint of 3–5 MB that was slow to release.

---

## Root cause

After `build_probability_base(df)` returned, `df` remained reachable until the enclosing scope exited naturally — which in a background thread could be delayed. No explicit `del df` was called, and `gc.collect()` was not invoked to force a collection cycle.

```python
# Before — df kept alive until scope exit, GC not forced
df = fetch_historical_data()
base = build_probability_base(df)
save_cache(base)
return base
```

The same pattern existed in both the synchronous (`load_or_build_base`) and asynchronous (`_async_rebuild`) rebuild paths.

---

## Fix

Explicit `del df` immediately after extraction, followed by `gc.collect()`, in both rebuild paths.

```python
# After
df = fetch_historical_data()
base = build_probability_base(df)
del df
gc.collect()
save_cache(base)
return base
```

Additionally, the exchange client was moved to a module-level singleton to eliminate per-call instantiation overhead.

---

## How to detect / prevent

- **Measure before and after.** VmPeak dropped from 1 222 MB to 284 MB (−77%). RSS at rest dropped from ~550 MB to ~62 MB. These numbers were only obtained by running `ps`/`/proc/<pid>/status` manually — there was no automated memory metric in place.
- **Know your rebuild schedule.** Any operation that allocates hundreds of MB on a timer (here: ~200–400 MB every 24h) must explicitly release that allocation. Don't rely on scope exit in a long-running process.
- **The async path is the dangerous one.** The synchronous path exits quickly; the daemon thread path (`_async_rebuild`) can hold references longer if the GC is not nudged. Audit both.
- **Don't remove the `del` + `gc.collect()` pair** without measuring the memory impact first — the saving is not theoretical.
- A secondary check: if the cache TTL expires while the circuit breaker is active (suppressing LLM calls entirely), `get_score_predictif()` is never called, the TTL check never runs, and the cache stays stale indefinitely. The refresh call must happen before the circuit breaker, not inside the LLM path.
