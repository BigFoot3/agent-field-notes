---
date: 2026-06-13
tags: systemd PrivateTmp IPC tmpfs two-services flag-file
one_liner: Two systemd services with PrivateTmp=yes each get a private /tmp namespace — a flag file written by one service is invisible to the other, silently breaking inter-process signaling.
---

# Two systemd services can't share `/tmp` — `PrivateTmp=yes` silently isolates IPC

**Severity:** INFO (feature dead on arrival — no data loss, no crash)
**System layer:** infra — systemd service isolation

---

## Symptom

The dashboard exposed a `/api/force_cycle` endpoint intended to trigger an immediate analysis cycle in the agent process. The endpoint returned HTTP 200 and appeared to succeed. The agent never responded. Forcing a cycle via the dashboard had no effect from day one, under any conditions.

No error was logged on either side.

---

## Diagnosis

The mechanism was a flag file: the dashboard wrote a file to `/tmp/force_cycle`, and the agent checked for its presence at the start of each cycle. If found, it would skip the normal sleep interval and run immediately.

Both services are declared in separate systemd unit files. Both unit files include `PrivateTmp=yes`.

`PrivateTmp=yes` is a systemd sandboxing directive that mounts a private `tmpfs` for the service at `/tmp` and `/var/tmp`. Each service gets its own isolated namespace. A file written to `/tmp` by the dashboard process exists in that service's private tmpfs — it is not visible in the agent's separate tmpfs, even though both paths are `/tmp/force_cycle` from the perspective of each process.

The two services never shared a filesystem at `/tmp`. The flag file was written and immediately discarded into the dashboard's private namespace.

---

## Root cause

`PrivateTmp=yes` is a standard hardening option, often added by default in service templates. Its effect on IPC is easy to overlook: any path under `/tmp` becomes service-local. There is no error, no warning, no `ENOENT` — the write succeeds, the read just never finds the file.

```ini
# Both unit files — each service gets its own /tmp
[Service]
PrivateTmp=yes
```

```python
# Dashboard — writes to its private /tmp
open("/tmp/force_cycle", "w").close()

# Agent — reads from its own private /tmp (different namespace)
if os.path.exists("/tmp/force_cycle"):   # always False
    skip_sleep()
```

---

## Fix

Move the flag file to an absolute path outside `/tmp` — specifically, a path within the project directory that both services can access on the real filesystem.

```python
# Dashboard
FORCE_CYCLE_FLAG = "/path/to/project/force_cycle.flag"
open(FORCE_CYCLE_FLAG, "w").close()

# Agent
if os.path.exists(FORCE_CYCLE_FLAG):
    os.remove(FORCE_CYCLE_FLAG)
    skip_sleep()
```

Both services have read/write access to the project directory, so the flag is now visible to both. `PrivateTmp=yes` can remain on both services — the fix does not weaken the sandbox.

---

## How to detect / prevent

- **Any `/tmp`-based IPC between two systemd services fails silently if either has `PrivateTmp=yes`.** Check unit files before designing flag-file, socket, or pipe mechanisms under `/tmp`.
- `systemd-analyze security <service>` lists active sandbox options including `PrivateTmp`. Run it on every service involved in IPC.
- A simple smoke test would have caught this immediately: write the flag, check its presence from the other service's process namespace (`nsenter` or a one-shot `ExecStartPre` that logs the directory listing).
- Prefer named sockets or a shared state file in a well-defined project directory over `/tmp` for cross-service signaling. The path should be explicit in both services' configurations.
