# Shared Memory

The shared memory directory is the nervous system of this setup. Both Sentinel (host) and your container agent read and write here. Local Claude on your dev machine reads it via SSH.

No message queues. No API calls between agents. Just files.

---

## Directory layout

```
/home/ubuntu/agents/memory/   ← host path
         ↕ bind mount
/app/memory/                    ← container path
```

Both paths point to the same directory on disk.

---

## Core files

### `sentinel-status.json`

Written by: **Sentinel** (every 15 min via `sentinel-status-update`)

```json
{
  "timestamp": "2026-03-01T14:02:00Z",
  "disk": {
    "used_gb": 10,
    "total_gb": 20,
    "pct": 51
  },
  "ram": {
    "used_mb": 1200,
    "total_mb": 1900,
    "free_mb": 700
  },
  "swap": {
    "used_mb": 400,
    "total_mb": 2048
  },
  "load": {
    "1min": 0.15,
    "5min": 0.12,
    "15min": 0.09
  },
  "uptime_seconds": 345600,
  "container": {
    "name": "your-agent-container",
    "status": "running",
    "started_at": "2026-02-28T10:00:00Z"
  }
}
```

### `hydra-state.json`

Written by: **Both agents**

Running event log. Auto-pruned to 24 hours / 200 events. Named "hydra" because the system has multiple heads.

```json
{
  "events": [
    {
      "timestamp": "2026-03-01T14:02:00Z",
      "agent": "sentinel",
      "event": "health_check",
      "data": {
        "disk_pct": 51,
        "swap_used_mb": 400,
        "container_status": "running",
        "alerts": []
      }
    },
    {
      "timestamp": "2026-03-01T14:15:00Z",
      "agent": "egger",
      "event": "monitor_run",
      "data": {
        "target": "your-project",
        "status": "ok",
        "items_checked": 12
      }
    }
  ]
}
```

### `sentinel-mailbox.json`

Written by: **Sentinel** (for the container agent to read)

```json
{
  "messages": [
    {
      "timestamp": "2026-03-01T22:00:00Z",
      "from": "sentinel",
      "message": "Tonight: audit the monitor logs from the past week and give me a summary."
    }
  ]
}
```

The container agent checks this on startup (configure in `BOOT.md`). After reading, it can clear processed messages or leave them for a few cycles.

### `egger-mailbox.json`

Written by: **Container agent** (for Sentinel or Local to read)

Same schema as `sentinel-mailbox.json`. The agent writes here to flag blockers, request restarts, or surface findings.

### `claude-usage.json`

Written by: **Sentinel** (optional, rolling window)

If you track Claude API spend, write rolling cost data here. Dashboard tools or Local Claude can read it.

```json
{
  "updated_at": "2026-03-01T14:30:00Z",
  "rolling_5h": {
    "input_tokens": 45000,
    "output_tokens": 12000,
    "cost_usd": 0.18
  },
  "weekly": {
    "input_tokens": 820000,
    "output_tokens": 210000,
    "cost_usd": 3.24
  }
}
```

---

## Agent-specific files

These live in the container's own workspace (`/app/workspace/memory/`), NOT in the shared directory. They're only readable via `docker exec`.

| File | Purpose |
|------|---------|
| `YYYY-MM-DD.md` | Daily running log (current session notes) |
| `journal.md` | Longer-form reflections, in-progress context |
| `your-project-state.json` | Per-project monitor state |
| `your-project-log.md` | Per-project monitor history |

Read them from Local:
```bash
ssh ubuntu@<your-ec2-ip> "docker exec your-agent-container cat /app/workspace/memory/YYYY-MM-DD.md"
```

---

## Avoiding write conflicts

Both agents write to the shared directory. To avoid conflicts:

- **Each agent owns separate files**. Sentinel writes `sentinel-status.json`; Egger writes `egger-mailbox.json`. They don't overwrite each other's files.
- **Hydra state uses atomic Python writes** (write to temp file, `mv` into place) — see `sentinel-check` script.
- **Mailboxes are append-only by convention**. Agents append messages, then prune old ones. Neither agent deletes messages the other hasn't read.

If you need tighter coordination, add a simple file lock:
```bash
flock /tmp/hydra.lock python3 update-hydra.py
```
