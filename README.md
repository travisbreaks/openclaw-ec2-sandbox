# OpenClaw on EC2 — Isolated Agent Setup

A pattern for running a persistent OpenClaw agent safely on a cheap EC2 server. The agent lives in Docker, isolated from the host. If it gets loose, you nuke the container. If the container isn't enough, you nuke the instance. You're out fifteen dollars a month and back up in twenty minutes.

---

## Why isolation matters

OpenClaw agents on uncontained machines have a reputation. File deletion. Prompt injection. If someone crafts the right input, or the agent interprets its instructions too liberally, the results range from expensive to catastrophic.

This setup puts the agent behind a Docker boundary. The host OS never sees its filesystem. Its RAM is capped. Its network is scoped. If something goes wrong, `docker stop` is a full stop.

The agent can still do real work: overnight research, monitoring jobs, GitHub crawling, writing reports you read in the morning. It just does that work from inside a box.

---

## Architecture

Two agents on one machine. A third on your dev box.

```
┌─────────────────────────────────────────────────────┐
│                   EC2 Instance                       │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Host OS (ubuntu)                           │   │
│  │                                             │   │
│  │  HOST AGENT (Claude Code)                   │   │
│  │  - Health monitoring (every 15 min)         │   │
│  │  - Docker lifecycle                         │   │
│  │  - Backups, security                        │   │
│  │  - Reads/writes ~/agents/memory/            │   │
│  │    (shared volume)                          │   │
│  │                                             │   │
│  │  ┌─────────────────────────────────────┐   │   │
│  │  │  Docker: agent container            │   │   │
│  │  │                                     │   │   │
│  │  │  CONTAINER AGENT (OpenClaw)         │   │   │
│  │  │  - Persistent workspace             │   │   │
│  │  │  - Overnight tasks, research        │   │   │
│  │  │  - Monitoring jobs                  │   │   │
│  │  │  - Reads/writes /app/memory/        │   │   │
│  │  │    (same shared volume, bind mount) │   │   │
│  │  └─────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
              ↕ SSH + file reads
┌─────────────────────────────────────────────────────┐
│  Local Dev Machine                                  │
│                                                     │
│  LOCAL CLAUDE CODE                                  │
│  - Reads container agent's memory files via SSH     │
│  - Ships code based on agent's findings             │
│  - Delegates tasks via shared mailbox               │
└─────────────────────────────────────────────────────┘
```

Three agents. One server. Call it what you want.

I call mine **Sentinel** (host) and **Egger** (container). The transmission linked at the bottom is the story of why.

---

## Component Roles

### Host agent

Runs as Claude Code directly on the EC2 instance. Owns the building.

- Monitors system health (disk, RAM, swap, load) on a cron
- Manages the Docker container lifecycle (start, stop, restart, gateway)
- Runs daily and weekly backups of Docker volumes and memory
- Does NOT own the container agent's world — it can restart the container but doesn't rewrite the agent's files

### Container agent

Runs inside Docker, isolated from the host. Has its own filesystem, its own workspace, its own memory.

- Persistent: Docker volume survives container restarts
- Gets tasks assigned via the shared mailbox
- Writes reports to memory files that Local picks up in the morning
- Has a defined identity: `SOUL.md`, `BOOT.md`, a voice, a purpose

### Local

Claude Code on your laptop or desktop. The third head.

- SSHes into the server to read the container agent's output
- Stages and ships code based on what the agent found
- Sends new tasks via `sentinel-msg`

---

## Shared Memory Pattern

The key to multi-agent coordination: a bind-mounted directory that both agents can read and write.

```
~/agents/memory/          ← host path (host agent writes here)
         ↕ bind mount
/app/memory/              ← container path (container agent writes here)
```

No API calls between agents. No message queues. Just files.

### Key files in shared memory

| File | Writer | Purpose |
|------|--------|---------|
| `sentinel-status.json` | Host agent (every 15min) | Disk, RAM, swap, load, container health |
| `hydra-state.json` | Both | Event log for all activity, auto-pruned |
| `sentinel-mailbox.json` | Host agent | Async messages to container agent |
| `egger-mailbox.json` | Container agent | Async messages back |
| `claude-usage.json` | Host agent | Rolling Claude API token/cost data |

See [memory/README.md](memory/README.md) for full schema documentation.

---

## Setup Guide

### Prerequisites

- EC2 instance (t3.small works, t3.medium is more comfortable)
- Ubuntu 22.04 or 24.04
- Docker + Docker Compose installed
- Anthropic API key
- Claude Code installed on host (`npm install -g @anthropic-ai/claude-code`)

### 1. Provision the server

```bash
# SSH in
ssh ubuntu@<your-ec2-ip>

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Install Claude Code
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt-get install -y nodejs
npm install -g @anthropic-ai/claude-code

# Set your API key
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.bashrc
source ~/.bashrc
```

### 2. Create directory structure

```bash
mkdir -p ~/agents/memory
mkdir -p ~/scripts
mkdir -p ~/backups/{daily,weekly}
```

### 3. Deploy the container

Copy the `docker/` directory from this repo to your server, fill in `.env`, then:

```bash
cd ~/agents
docker compose up -d
```

The container uses a named volume (`agent-data`) for persistence. The shared memory directory is bind-mounted.

### 4. Install the scripts

Copy the `scripts/` directory to `~/scripts/` on the server. Make them executable:

```bash
chmod +x ~/scripts/sentinel-*
chmod +x ~/scripts/backup-volumes
```

### 5. Set up cron

```bash
crontab -e
```

Add:

```
# Health check (every 15 min)
*/15 * * * * /home/ubuntu/scripts/sentinel-check >> /var/log/sentinel.log 2>&1

# Status update (every 15 min, offset by 2)
2,17,32,47 * * * * /home/ubuntu/scripts/sentinel-status-update >> /var/log/sentinel-status.log 2>&1

# Daily backup (02:00 UTC)
0 2 * * * /home/ubuntu/scripts/backup-volumes >> /var/log/backup.log 2>&1

# Weekly backup (Sunday 03:00 UTC)
0 3 * * 0 /home/ubuntu/scripts/backup-volumes --weekly >> /var/log/backup.log 2>&1
```

**Note**: Cron runs with a stripped PATH. Always use absolute paths in scripts.

### 6. Configure your agent

Copy `agent/SOUL.md.example` into your container's workspace as `SOUL.md`. Fill it in. Same for `BOOT.md.example`. These tell your agent who it is and what to do when it wakes up.

### 7. Send your first message

```bash
./scripts/sentinel-msg "Hey — you're live. Check in when you're ready."
```

---

## Scripts Reference

| Script | What it does |
|--------|-------------|
| `sentinel-check` | Checks disk, RAM, swap, and container health. Writes to hydra-state.json. |
| `sentinel-status-update` | Writes a full status JSON (disk, RAM, swap, load, container health, timestamp). |
| `sentinel-msg` | Sends a message to the container agent via the shared mailbox. |
| `backup-volumes` | Backs up Docker volumes, shared memory, and scripts. Supports `--weekly` flag. |

---

## What Breaks

**Memory exhaustion**: t3.small has 2GB RAM. With 2GB swap, you have 4GB total. Docker container at 1.5GB limit + host cron scripts + VS Code Remote SSH = tight. If swap hits 98%, SSH daemon can't accept new connections. The OS is fine; userspace is frozen. Fix: stop the instance from the AWS console, wait for "stopped", then start it. Elastic IP means same address. Docker auto-restarts the container.

**Cron PATH**: bare script names fail silently in cron. Always use absolute paths.

**Shared volume conflicts**: if both agents write to the same key in the same file at the same time, you'll get corruption. Use separate files for each agent's writes, or implement simple file locking.

**Context evaporation**: Claude Code sessions don't persist across restarts by default. Write a session file (`sentinel-session.json`) that the host agent reads on boot. Include: current task, in-flight work, pending follow-ups.

---

## What it Enables

- **Overnight research**: assign a task before bed, read the report in the morning
- **Passive monitoring**: container agent watches things on a schedule, alerts via mailbox
- **Async handoffs**: agent writes a bug report; Local picks it up at next session
- **Running context**: both agents accumulate memory over weeks, not just one session

---

## The story behind this

[Transmission #053: SENTINEL & EGGER](https://travisbreaks.org/transmissions/053-sentinel-and-egger/) — the narrative of how this setup came to be, including the time swap hit 98% and SSH froze, and what it was like to read a report that opened with "Mode: sleeping."

---

## License

MIT
