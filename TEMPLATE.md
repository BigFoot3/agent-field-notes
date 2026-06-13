# [Short title — searchable, describes what broke]

**Tags:** `[technology tags]` `[symptom tags]`
**Severity:** CRITICAL / HIGH / MEDIUM / LOW
**System layer:** agent loop / exchange integration / dashboard / state / infra

---

## Symptom

What was observed in production. How it manifested.
What logs / alerts surfaced (or didn't).

---

## Diagnosis

How the bug was found. What made it hard to spot.
Any misleading signals.

---

## Root cause

The actual line(s) of code, the invariant violated, the assumption that was wrong.

```python
# Minimal reproduction — stripped of project-specific context
```

---

## Fix

What changed. Why this fix and not another.

```python
# After
```

---

## How to detect / prevent

- What monitoring would have caught this earlier
- What test now covers this case
- What invariant or review checklist prevents recurrence
