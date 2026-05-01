# breadAI

**AI that audits your infrastructure. Then a second AI audits the first one.**

A 26,000-line Python platform that monitors ~100 devices across 6 VLANs, compresses 11M+ characters of daily telemetry down to ~2,800 (99.97% reduction), and delivers a single actionable morning report via Discord. The headline feature: **CLIde** — after the primary Claude audit produces findings, a second Claude instance with live SSH/curl/bash access independently verifies every claim against real device state. Hallucinations, stale data, and false positives get caught before the report is sent.

~26,000 lines of Python | 23 modules | 10-phase audit pipeline | AI-verifying-AI architecture

---

## What It Does

Every morning at 8 AM, breadAI:

1. **Collects** data from 10 syslog databases, 3 Pi-hole DNS servers, 2 Proxmox hypervisors, pfSense firewall, Brocade managed switch, UniFi wireless controller, Emby media server, Proxmox Backup Server, SabNZBd, Uptime Kuma, Nginx Proxy Manager, and Docker/Portainer
2. **Compresses** ~11 million characters of raw syslog down to ~2,800 characters (~99.97% reduction) using a multi-stage pipeline: regex noise filtering, severity tagging, smart dedup, adaptive scoring, and local LLM summarization
3. **Analyzes** the compressed data with Claude (Anthropic API) using a model cascade (Sonnet &rarr; Opus &rarr; Haiku fallback)
4. **Verifies** the AI's claims by dispatching a second Claude instance (CLIde) that can SSH into live devices, query databases, and curl APIs to independently confirm or contradict each finding
5. **Reports** via a color-coded Discord embed with WAN health, bandwidth, infrastructure status, downtime events, and an AI-generated analysis

It also serves as an interactive network analyst via Discord `@mention` with 17 chat tools for on-demand queries.

---

## Architecture

### 10-Phase Audit Pipeline

```
generate_report()
    |
    v
[1] Setup .............. Load config, chat history, memory rules, init analyzers
[2] Collection ......... Mount verification, 10 parallel DB scans, NSFW detection
[3] Telemetry .......... Pi-hole stats, RRD latency/loss, SNMP bandwidth, anomaly alerts
[4] API Fetches ........ 8 parallel API calls (PBS, PVE x2, SabNZBd, UniFi, Emby, Kuma, Docker)
[5] Compression ........ Noise filter -> severity tag -> dedup -> Ollama LLM summarize
[6] Integrity .......... Compression ratio validation, 22-point system self-check
[7] AI Audit ........... Claude API analysis (model cascade with automatic fallback)
[8] Verification ....... CLIde second-pass: live SSH/curl fact-checking of flagged findings
[9] Embed Build ........ Discord embed construction, color escalation logic
[10] Send + Cleanup .... Deliver report, persist 30-day history, auto-resolve stale alerts
```

All 10 phases share an `AuditData` dataclass (~120 fields) passed by reference as the state carrier.

### CLIde: AI-Verifying-AI

The most architecturally interesting component. After the primary Claude audit produces findings, breadAI:

1. Extracts structured findings from the audit (DB failures, infrastructure warnings, NSFW alerts, threshold breaches, anomalies)
2. Builds targeted verification prompts with specific "HOW to verify" hints (SSH commands, API endpoints, file paths)
3. Dispatches a second Claude instance running on a separate LXC container with full bash/SSH/curl tool access
4. CLIde independently verifies each finding against the live network state
5. Results appear in the Discord embed as "CLIde's Review" with per-finding verdicts. All-clear results get a short personality response; contradictions get full detail.

This catches hallucinations, stale data, and false positives that a single-pass analysis would miss.

### Compression Pipeline

Raw syslog data goes through 5 stages before reaching Claude:

```
11,176,305 chars (raw)
       |
  [Regex Noise Filter] .......... 40+ patterns strip known-noise lines
       |
  [Severity Tagging] ............ [ERR] / [WARN] / [INFO] classification
       |
  [Smart Dedup + Bucketing] ..... Adaptive scoring: log(freq) + severity + novelty(sigma)
       |
  [Local LLM Summarization] ..... Ollama (llama3.1 on RTX 4000 8GB)
       |
     2,806 chars (compressed) .... 99.97% reduction
```

This ensures Claude sees only actionable signal, not raw noise.

### Dual-WAN Health Scoring

Each WAN link (10G fiber primary + fiber backup) is monitored through SmokePing with 3+ external targets per link. Health assessment uses majority-vote scoring:

- If 2+ of 3 external targets show degradation &rarr; **WAN issue** (ISP problem)
- If only 1 target degrades &rarr; **target issue** (remote server problem)
- Gateway-only latency without external confirmation &rarr; suppressed (local noise)

This eliminates false WAN alerts from individual target flaps.

---

## Key Engineering Decisions

### Model Cascade with Cost Optimization
The audit tries Sonnet first (~$0.04/audit). If Sonnet fails (rate limit, capacity), it falls back to Opus, then Haiku. CLIde verification runs Sonnet with `CLAUDE_CODE_SIMPLE=1` to strip unnecessary context, reducing per-verification cost from $0.15 to ~$0.03.

### Anomaly Detection
Statistical anomaly detection using 3-sigma analysis against rolling 30-day baselines. Seven independent historical datasets track latency, DNS queries, firewall blocks, backup sizes, VM resource usage, media activity, and alert frequency. Threshold alerts fire for absolute limits; anomaly alerts fire for statistical deviations.

### Alert Fatigue Prevention
Extensive noise suppression at every layer:
- 40+ regex patterns filter before LLM processing
- Firewall blocks from internet are always suppressed (real attacks manifest as latency degradation, not block counts)
- QUIC/HTTP3 bidirectional blocks recognized as normal return traffic
- Brief downtime blips filtered from reports
- Sub-0.5% packet loss over 24h suppressed (1-2 dropped pings across a day is noise)
- Alert tracking with auto-resolution prevents stale warnings

### Chat Intelligence
The bot responds to `@mention` queries with 17 tools including syslog search, device lookup, backup status, forecast generation, traffic analysis, and historical metrics. Chat context includes conversation history, device registry, and compressed recent audit data. An Ollama compression step keeps chat context within token limits using a bookend strategy (first 1/3 + last 2/3 preserves newest data).

### Verifier Guardrails (CLIde)
CLIde was originally given `--max-turns 25` and a polite "please do this in a single bash call" system prompt. Result: ~40% failure rate &mdash; it would ignore the hint, run one command per finding, hit the timeout, or exhaust turns without answering. The fix was to remove its ability to misbehave, not ask it nicely: `--max-turns 3` (one bash call + one answer, hard cap), a directive system prompt ("You have 1 tool call"), and a tool allowlist of just `Bash` (no file reads). Making the constraint physical instead of verbal dropped failures to near zero.

### Platform-Specific Quirks
Real hardware is weirder than documentation. A few that cost debugging time:
- **Brocade ICX switches** reject SSH `exec_command()` with "Channel closed" &mdash; they only accept interactive shell channels. Fix: `invoke_shell()` + buffered reads + `skip-page-display`.
- **SQLite over NFS** with WAL mode fails "file is not a database" when the shared-memory region can't be mmap'd remotely. Fix: try-query, fall back to `immutable=1` URI (skips WAL, reads last checkpoint).
- **Claude Code CLI** hangs on stdin file redirect if the input contains Unicode (emoji, bullets). Fix: strip to ASCII before writing the temp file.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Runtime** | Python 3.11, Docker, Synology NAS |
| **AI / LLM** | Claude API (Anthropic), Ollama + llama3.1 (local, RTX 4000 8GB) |
| **Bot Framework** | discord.py 2.x |
| **Network Monitoring** | SmokePing (RRD), Pi-hole (3 instances), pfSense (SNMP + SQLite) |
| **Infrastructure** | Proxmox VE (2 hypervisors), Proxmox Backup Server, Portainer |
| **Switching** | Brocade ICX7250 (SSH MAC table validation) |
| **Wireless** | UniFi controller API |
| **Media** | Emby API (watch history, session tracking) |
| **Storage** | PostgreSQL 16 (structured metrics), JSON (30-day rolling history) |
| **Connectivity** | Paramiko (SSH), SNMP, REST APIs, SQLite (syslog DBs) |

---

## Module Map

```
breadai/
  main.py .................. Discord bot entry + 17 chat tools + context builder
  ai_orchestration.py ...... 10-phase audit pipeline (generate_report + AuditData)
  ai.py .................... Ollama compression, severity tagging, scrubbing
  scanners.py .............. Data collection: DBs, Pi-hole, NSFW, Emby, PBS, PVE,
                             UniFi, SabNZBd, Brocade, pfSense SNMP, NPM, Docker
  analyzers.py ............. DeviceRegistry, TrafficAnalyzer, EventCorrelator
  storage.py ............... JSON persistence + 30-day history management
  config.py ................ Centralized config, syslog safeguards, noise patterns
  commands.py .............. 12 Discord !commands (!audit, !status, !forecast, etc.)
  anomaly_detection.py ..... 3-sigma statistical anomaly detection
  threshold_alerts.py ...... Real-time threshold monitoring
  analytics.py ............. Math utilities (regression, confidence intervals)
  analytics_forecast.py .... 7 forecasting functions (storage, latency, RAM, etc.)
  analytics_patterns.py .... 10 pattern analysis functions (traffic, correlations)
  backup_monitor.py ........ Backup status verification
  models.py ................ Pydantic data models
  db_pool.py ............... PostgreSQL connection pooling
  storage_db_unified.py .... SQL-first with JSON backup (dual-write)
  web_server.py ............ Health check endpoint for Docker
```

---

## Discord Commands

| Command | Description |
|---------|-------------|
| `!audit` | Trigger a full audit cycle on demand |
| `!status` | Quick system health summary |
| `!forecast` | Predictive analytics (storage, latency, RAM, firewall trends) |
| `!devices` | Device registry with IP/MAC/VLAN mapping |
| `!ports` | Live Brocade switch port map vs expected topology |
| `!backup` | Proxmox Backup Server status |
| `!pihole` | Pi-hole DNS statistics across all 3 instances |
| `!emby` | Recent media watch activity |
| `!noise` | Manage custom noise suppression patterns |
| `!about` | System info and monitored infrastructure |
| `@breadAI` | Natural language queries with full tool access |

---

## Monitored Infrastructure

- **Network**: Dual-WAN gateway, managed L3 switch, 4 wireless APs, 6 VLANs
- **Compute**: 2 Proxmox hypervisors running ~15 VMs/CTs
- **Storage**: Synology NAS (primary), Proxmox Backup Server, Backblaze B2 (offsite)
- **DNS**: 3 Pi-hole instances (distributed across hypervisors + NAS)
- **Services**: Emby, SabNZBd, Sonarr, Radarr, Nginx Proxy Manager, Uptime Kuma
- **Logging**: Centralized syslog (10 databases), SmokePing RRD telemetry
- **Security**: pfSense firewall (pfBlockerNG), NSFW domain detection, Frigate NVR

---

## Development Approach

This project is built and maintained using **Claude Code** (Anthropic's CLI agent) as a pair-programming partner. Claude Code reads the full codebase, SSH's into live infrastructure for debugging, and has project memory files that persist architecture decisions and operational lessons across sessions.

Deployment moved from direct-push to GitOps during v0.5.5c: a private stack repo is watched by [doco-cd](https://github.com/kimdre/doco-cd), which polls every 5 minutes and redeploys whichever stacks changed. Any container-level runtime tweak that doesn't survive a recreate is now a bug &mdash; config has to live in the compose, the image, or a mounted volume.

All non-obvious code decisions include inline `# WHY` comments so the AI assistant (and future human readers) understand the reasoning behind implementation choices.

---

## Status

**v0.5.5c** &mdash; Active, running daily audits since December 2025. ~26,000 lines of Python across 23 modules.

This is a private infrastructure project. Source code is not published due to embedded network-specific configuration, but the architecture and engineering patterns are documented here for portfolio reference.

---

*Built with Python, Claude, and an unreasonable amount of syslog data.*
