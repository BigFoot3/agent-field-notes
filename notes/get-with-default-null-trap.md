---
date: 2026-06-13
tags: python dict.get null-default ccxt exchange-api silent-bug
one_liner: dict.get(key, fallback) does not trigger when the key exists with a zero value — the fallback is silently skipped, returning 0.0 instead of the computed estimate.
---

# `dict.get(key, default)` silent failure when key exists with a zero value

**Severity:** LOW (data integrity — no order loss, accounting impact)
**System layer:** exchange integration — sell order result parsing

---

## Symptom

After a successful sell order, the amount of stablecoins received was recorded as `0.0` in the trade result. Downstream code that read this field logged a misleading value. No exception was raised, no alert fired. The order itself executed correctly on the exchange side.

---

## Diagnosis

The exchange API response for a market sell order includes a `cost` field representing the total stablecoin proceeds. In most cases this field is populated correctly. However, in some responses the exchange returns `cost` with an explicit value of `0` — the field is present in the dict, but its numeric value is zero.

The code used `order.get("cost", fallback)` to extract this value. Python's `dict.get` only uses the default when the key is **absent**. When the key is present with a value of `0`, `get` returns `0` — the fallback is never evaluated.

```python
# Before — fallback not triggered when cost=0 (key present, value falsy)
usdt_received = float(order.get("cost", btc_to_sell * exec_price))
# → returns 0.0 if the exchange returns {"cost": 0, ...}
```

This is a well-known Python footgun but easy to miss when the falsy case (zero) looks like legitimate data in other contexts.

---

## Root cause

`dict.get(key, default)` semantics: the default is returned only on **KeyError**, not on falsy values. The exchange occasionally returns `cost: 0` in the order response — the field exists, so `get` returns `0` and the fallback computation `btc_to_sell * exec_price` is never reached.

The correct guard for "absent or falsy" is the `or` operator:

```python
# After — fallback triggers on both absent key and falsy value
usdt_received = float(order.get("cost") or (btc_to_sell * exec_price))
```

`order.get("cost")` returns `None` if the key is absent, or the value if present. The `or` then evaluates the right-hand side if the result is falsy — covering both the missing-key and zero-value cases.

---

## How to detect / prevent

- **Audit every `dict.get(key, non_trivial_default)` call** in exchange API response parsing. If the fallback is a computed estimate (not just `None` or `0`), the `or` pattern is almost always more correct.
- **Distinguish two different intents:**
  - "Use default if key is missing" → `dict.get(key, default)` is correct
  - "Use default if key is missing or value is falsy" → `dict.get(key) or default` is correct
- **Test with a mocked response where the field is `0`**, not just absent. A test that only checks the absent-key case will pass on the buggy code.
- The same pattern is worth checking in any integration that parses third-party API responses where numeric fields may be present but zero due to exchange-side edge cases.
