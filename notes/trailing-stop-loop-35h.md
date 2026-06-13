---
date: 2026-06-13
tags: trailing-stop ccxt balance-desync fee-precision infinite-loop exchange
one_liner: A sub-satoshi discrepancy between the locally tracked BTC amount and the actual exchange balance caused the trailing stop to attempt the same failing sell order every 5 minutes for 35 hours.
---

# Trailing stop firing in a loop — ccxt insufficient balance, 35 hours

**Severity:** CRITICAL (live orders blocked; position stuck open during full stop duration)
**System layer:** exchange integration — sell order execution + balance sync

---

## Symptom

The trailing stop fired correctly when the price threshold was breached. The agent called `sell_btc()` with the locally tracked quantity. The exchange rejected the order with an insufficient balance error. The agent logged the failure, did nothing to reconcile the state, and retried at the next 5-minute stop-check cycle. This repeated every 5 minutes for approximately 35 hours until the issue was manually diagnosed and patched.

No alert fired for the repeated failures. The position remained open throughout.

---

## Diagnosis

The locally tracked BTC quantity (`btc_held`) was `0.00111`. The actual free balance on the exchange (`btc_free`) was `0.001109`. A difference of `0.000001 BTC` — below any meaningful threshold, but enough for the exchange to reject the order.

The discrepancy originated at buy time. The exchange's fee structure allows fees to be deducted in the base asset (BTC) rather than the quote asset, depending on account configuration. When this mode is active, `order["filled"]` represents the gross quantity matched — the fee has already been taken from that amount. Recording `order["filled"]` directly as `btc_bought` overstated the actual balance by exactly the fee amount.

```python
# Before — records gross filled quantity, fee not deducted
btc_bought = float(order["filled"])
```

On the sell side, the failure handler only checked for dust (balance below `0.0001 BTC`). It did not handle the case where `btc_free < btc_held` — a real position with a precision mismatch.

```python
# Before — sync only handled dust, not precision divergence
if btc_free < 0.0001:
    portfolio["btc_held"] = 0.0  # treat as zero
# btc_free = 0.001109, btc_held = 0.00111 → neither branch matched → no retry
```

---

## Root cause

Two compounding issues:

1. **Buy path:** fee deduction not applied when the exchange charges fees in BTC.
2. **Sell path:** failure handler did not cover the `btc_free < btc_held` case, so no reconciliation occurred.

The combination meant every trailing stop attempt used a quantity the exchange would always reject, with no mechanism to self-correct.

---

## Fix

**Buy path** — inspect the fee currency and subtract if applicable:

```python
fee_info = order.get("fee") or {}
btc_fee  = float(fee_info.get("cost", 0)) if fee_info.get("currency") == "BTC" else 0.0
btc_net  = float(order.get("filled", btc_quantity)) - btc_fee
portfolio["btc_held"] = btc_net
```

**Sell failure handler** — extend sync logic to cover precision divergence:

```python
if btc_free < 0.0001:
    # Genuine dust — reset position
    portfolio["btc_held"] = 0.0
elif btc_free < portfolio["btc_held"]:
    # Precision mismatch — correct and retry once
    portfolio["btc_held"] = btc_free
    result = sell_btc(btc_amount=btc_free)
# else: transient failure (network etc.) — do not modify state
```

The retry is bounded to one attempt. If it fails again, the cycle exits without modifying the portfolio, avoiding a reset that would destroy the position's entry price and peak — which would cause the reconciliation logic to re-adopt the position at the current price, ratcheting the trailing stop downward.

---

## How to detect / prevent

- **Log every sell failure with the attempted quantity and the exchange balance.** The mismatch was present from the first attempt but would only be visible if both values were logged together.
- **Test the fee-in-base-asset path explicitly.** Mock `order["fee"] = {"currency": "BTC", "cost": 0.000001}` and assert that `btc_held` reflects the net quantity.
- **Alert on repeated sell failures.** A single failure can be transient. Two consecutive failures on the same order warrant a notification — 35 hours of silent retries is not acceptable.
- **Never reset position state on a failed sell unless the balance is confirmed dust.** A blind reset when `btc_free >= 0.0001` destroys entry price, peak, and trailing stop anchor — the downstream reconciliation will re-adopt the position at current price, silently degrading the stop.
