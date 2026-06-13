# agent-field-notes

Real incident reports from a long-running LLM + exchange trading system in production.

Not glamorous. Not a success story. Just what actually breaks — and why.

---

## What this is

A public archive of engineering incidents from a production system combining:
- A polling agent (25–50 min cycles) calling Claude Haiku/Sonnet for decisions
- A live exchange integration (ccxt) handling real orders
- A Flask dashboard served via systemd + nginx
- Several JSON files as shared state, protected by `fcntl` locks and atomic writes
- A real-time crash detector via WebSocket

These notes are reconstructed from actual git history, changelogs, and production logs.
They are anonymized: no IPs, no capital figures, no trading performance.

---

## Why

Because "LLM agent in production" posts usually show the architecture diagram.
They skip the part where:
- a trailing stop fires in a loop for 35 hours
- the LLM returns a field as `None` instead of `0` and crashes the dashboard
- a `get(key, default)` silently passes when the field exists but is null
- two systemd services can't share `/tmp` due to `PrivateTmp=yes`

That's the part that takes time. That's what these notes cover.

---

## Index

4 field notes.

| Note | One-liner |
|------|-----------|
| [Pandas DataFrame not freed between cycles](notes/pandas-memory-leak-24h-cycle.md) | A pandas DataFrame held alive across a 24h cache rebuild cycle pushed RSS from ~62 MB to 1.2 GB on a constrained VPS. |
| [`dict.get(key, default)` silent failure when key exists with a zero value](notes/get-with-default-null-trap.md) | `dict.get(key, fallback)` does not trigger when the key exists with a zero value — the fallback is silently skipped, returning 0.0 instead of the computed estimate. |
| [Two systemd services can't share `/tmp` — `PrivateTmp=yes` silently isolates IPC](notes/privatetmp-ipc-broken.md) | Two systemd services with `PrivateTmp=yes` each get a private `/tmp` namespace — a flag file written by one service is invisible to the other, silently breaking inter-process signaling. |
| [Trailing stop firing in a loop — ccxt insufficient balance, 35 hours](notes/trailing-stop-loop-35h.md) | A sub-satoshi discrepancy between the locally tracked BTC amount and the actual exchange balance caused the trailing stop to attempt the same failing sell order every 5 minutes for 35 hours. |

---

## Format

Each note follows the same structure — see [TEMPLATE.md](TEMPLATE.md).
