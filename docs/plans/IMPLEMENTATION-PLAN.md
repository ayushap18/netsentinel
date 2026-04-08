# NetSentinel — Implementation Plan

**Version:** 1.0
**Date:** 2026-04-09

---

## Phase Overview

```
Phase 1: Foundation          ██████░░░░░░░░░░░░░░  (Weeks 1-3)
Phase 2: Analysis Pipeline   ████████░░░░░░░░░░░░  (Weeks 4-6)
Phase 3: Web Dashboard       ██████████░░░░░░░░░░  (Weeks 7-8)
Phase 4: CLI + Alerts        ████████████░░░░░░░░  (Weeks 9-10)
Phase 5: ML + Intelligence   ██████████████░░░░░░  (Weeks 11-12)
Phase 6: Hardening           ████████████████░░░░  (Week 13)
```

---

## Phase 1: Foundation (Weeks 1-3)

**Goal:** Core server running, basic agent capturing packets, database storing data, WebSocket streaming.

### Week 1: Project Setup + Database

| Task | Description | Output |
|------|-------------|--------|
| 1.1 | Initialize monorepo structure (pyproject.toml, package.json, Docker) | Project skeleton |
| 1.2 | Set up Docker Compose (TimescaleDB, Redis, Core placeholder) | `docker-compose.yml` |
| 1.3 | Create database schema + Alembic migrations | All tables created |
| 1.4 | Set up SQLAlchemy models + Pydantic schemas | `models/`, `schemas/` |
| 1.5 | Basic FastAPI app with health endpoint | Core boots, `/api/health` works |
| 1.6 | Redis connection + basic device state helpers | Redis read/write working |

**Milestone:** `docker-compose up` starts all services, Core serves health endpoint.

### Week 2: Agent + Agent-Core Communication

| Task | Description | Output |
|------|-------------|--------|
| 2.1 | Agent skeleton (config, CLI entry point via Typer) | `netsen-agent setup/start/stop` |
| 2.2 | WebSocket server in Core (Agent Manager) | Core accepts WS connections |
| 2.3 | WebSocket client in Agent (MessagePack, TLS, reconnect) | Agent connects to Core |
| 2.4 | Heartbeat system (Agent sends, Core tracks device status) | Devices go online/offline |
| 2.5 | Agent authentication (API key in WS handshake) | Unauthorized agents rejected |
| 2.6 | Device registration flow (first connect = auto-register) | Devices appear in DB |

**Milestone:** Agent connects to Core, sends heartbeats, device shows as "online" in DB + Redis.

### Week 3: Packet Capture + Basic Ingestion

| Task | Description | Output |
|------|-------------|--------|
| 3.1 | Scapy capture module (laptop) | Raw packets captured |
| 3.2 | tcpdump capture module (Android/Termux) | Packets captured on Android |
| 3.3 | VPN-TUN capture module (no-root Android) | Non-root capture working |
| 3.4 | Local packet filtering (drop noise) | Reduced data volume |
| 3.5 | Packet serialization + streaming to Core | Packets arrive at Core |
| 3.6 | Core: L1 Packet Engine — parse, store in TimescaleDB | Packets in database |
| 3.7 | Basic REST endpoints: GET /devices, GET /devices/{id}/packets | API returns packet data |

**Milestone:** End-to-end: Agent captures packet -> sends to Core -> stored in DB -> queryable via API.

---

## Phase 2: Analysis Pipeline (Weeks 4-6)

**Goal:** Full 5-layer pipeline operational. Flows assembled, apps classified, threats detected, responses executed.

### Week 4: Flow Analysis + App Classification (L2 + L3)

| Task | Description | Output |
|------|-------------|--------|
| 4.1 | L2: Flow assembly engine (TCP session tracking, UDP grouping) | Packets grouped into flows |
| 4.2 | L2: Flow table in Redis (active flows) + persist to TimescaleDB | Flows queryable |
| 4.3 | L2: Flow metrics (bytes, packets, RTT, retransmissions) | Rich flow metadata |
| 4.4 | L3: Port-based app identification | Known ports mapped to apps |
| 4.5 | L3: DNS correlation (match DNS queries to flow destinations) | Domain names on flows |
| 4.6 | L3: TLS SNI extraction for HTTPS flows | HTTPS flows identified |
| 4.7 | GeoIP integration (MaxMind GeoLite2) | Country/city on flows |
| 4.8 | REST endpoints: GET /devices/{id}/flows, GET /stats/apps | Flow + app data via API |

**Milestone:** Flows assembled with app names, domains, and geo data visible via API.

### Week 5: Threat Detection (L4)

| Task | Description | Output |
|------|-------------|--------|
| 5.1 | Rule engine framework (load YAML, evaluate conditions) | Rule engine operational |
| 5.2 | Default rule set (20+ rules for common threats) | `rules/default_rules.yaml` |
| 5.3 | IP/domain blocklist matching (threat feed integration) | Malicious IP detection |
| 5.4 | DNS entropy analysis (tunneling detection) | DNS tunnel detection |
| 5.5 | DPI pattern matching (plaintext credentials, etc.) | Application-layer detection |
| 5.6 | ARP spoof detection (duplicate IP-MAC mappings) | WiFi attack detection |
| 5.7 | Threat feed downloader (abuse.ch, AlienVault OTX) | Auto-updated threat intel |
| 5.8 | Threat storage + REST endpoints | Threats in DB + API |

**Milestone:** Simulated attacks (connect to known-bad IP, DNS tunnel) trigger threat detections.

### Week 6: Tiered Response (L5) + Agent Enforcement

| Task | Description | Output |
|------|-------------|--------|
| 6.1 | Response engine (evaluate severity -> choose action) | Tiered response working |
| 6.2 | LOW response: log to DB, publish to WebSocket | Silent logging |
| 6.3 | MEDIUM response: log + prepare alert payload | Alert payloads generated |
| 6.4 | HIGH response: log + send BLOCK command to agent | Auto-blocking |
| 6.5 | CRITICAL response: log + send QUARANTINE command | Device quarantine |
| 6.6 | Agent enforcer: iptables/nftables block execution | IP blocking on device |
| 6.7 | Agent enforcer: quarantine mode (block all non-Core traffic) | Device isolation |
| 6.8 | Forensic snapshot saving (dump flagged packets to .pcap) | Evidence preservation |
| 6.9 | Blocklist/whitelist management endpoints | Manual block/allow |

**Milestone:** Full pipeline: packet -> flow -> classify -> detect threat -> auto-block + save forensics.

---

## Phase 3: Web Dashboard (Weeks 7-8)

**Goal:** Fully functional real-time web NOC dashboard.

### Week 7: Dashboard Foundation + Core Pages

| Task | Description | Output |
|------|-------------|--------|
| 7.1 | Next.js project setup (TypeScript, Tailwind, project structure) | Dashboard skeleton |
| 7.2 | Auth page (JWT login) | Login working |
| 7.3 | WebSocket hooks (useWebSocket, useDeviceData, useThreatStream) | Real-time data hooks |
| 7.4 | API client library (typed fetch wrapper) | `lib/api.ts` |
| 7.5 | Layout: sidebar nav, header with global stats | App shell |
| 7.6 | Overview page: device grid, bandwidth gauge, threat count | Main dashboard |
| 7.7 | Device detail page: info card, app breakdown pie chart | Device drill-down |

**Milestone:** Dashboard shows live device status, bandwidth, and per-device app breakdown.

### Week 8: Advanced Visualizations + Threat Center

| Task | Description | Output |
|------|-------------|--------|
| 8.1 | Device detail: flow table (sortable, filterable) | Connection browser |
| 8.2 | Device detail: packet inspector (click flow -> see packets) | Packet drill-down |
| 8.3 | Traffic page: bandwidth over time chart (Recharts) | Time-series graphs |
| 8.4 | Traffic page: top talkers table | Top destinations |
| 8.5 | Traffic page: geographic map (MapLibre + GeoIP data) | World map visualization |
| 8.6 | Threat center: threat list with severity colors | Threat dashboard |
| 8.7 | Threat center: one-click actions (block, whitelist, dismiss) | Interactive response |
| 8.8 | Threat center: forensic detail view | Evidence viewer |
| 8.9 | Settings page: device management, rule editor | Configuration UI |
| 8.10 | Real-time updates: toast notifications for new threats | Live alerts |

**Milestone:** Complete NOC dashboard with real-time visualization, device drill-down, and threat management.

---

## Phase 4: CLI + Alerts (Weeks 9-10)

**Goal:** CLI tool and Telegram bot fully operational.

### Week 9: CLI Tool

| Task | Description | Output |
|------|-------------|--------|
| 9.1 | CLI skeleton (Typer + Rich, config file, API client) | `netsen` command works |
| 9.2 | `netsen devices` — device listing with status table | Device overview |
| 9.3 | `netsen device <id>` — detailed device info | Device inspection |
| 9.4 | `netsen flows <device>` — flow listing with filters | Flow query |
| 9.5 | `netsen packets <device>` — packet query + export | Packet inspection |
| 9.6 | `netsen monitor` — live TUI dashboard (Rich Live) | Real-time terminal UI |
| 9.7 | `netsen threats` — threat listing and actions | Threat management |
| 9.8 | `netsen stats` — bandwidth and app statistics | Stats reporting |
| 9.9 | `netsen block/whitelist` — manual IP/domain management | Block/allow |

**Milestone:** Full CLI parity with dashboard functionality.

### Week 10: Telegram Bot + Alert System

| Task | Description | Output |
|------|-------------|--------|
| 10.1 | Telegram bot setup (python-telegram-bot, webhook/polling) | Bot online |
| 10.2 | Alert service in Core (receives threat events, dispatches) | Alert pipeline |
| 10.3 | Severity-based notification routing | Right alerts to right channel |
| 10.4 | Rich alert messages (severity icon, device, IP, description) | Clear notifications |
| 10.5 | Inline action buttons ([Block] [Whitelist] [Dismiss]) | Interactive alerts |
| 10.6 | Bot commands: /status, /devices, /threats | Quick checks from phone |
| 10.7 | /block, /whitelist, /mute commands | Actions from phone |
| 10.8 | Daily summary report (scheduled, sent every morning) | Digest notifications |
| 10.9 | Notification preferences (severity filter, quiet hours) | User config |

**Milestone:** Telegram alerts arrive within 5 seconds of threat detection, with actionable buttons.

---

## Phase 5: ML + Intelligence (Weeks 11-12)

**Goal:** ML-powered traffic classification and anomaly detection operational.

### Week 11: Traffic Classifier

| Task | Description | Output |
|------|-------------|--------|
| 11.1 | Data collection script (capture labeled traffic from own devices) | Training dataset |
| 11.2 | Feature engineering (packet size dist, inter-arrival, port, TLS FP) | Feature pipeline |
| 11.3 | Train Random Forest / XGBoost classifier | Model artifact |
| 11.4 | Integrate classifier into L3 pipeline | ML classification live |
| 11.5 | Evaluation metrics + confusion matrix | Model performance report |
| 11.6 | Fallback chain: ML -> DPI -> DNS -> port (graceful degradation) | Robust classification |

**Milestone:** Unknown traffic flows classified by ML with >80% accuracy.

### Week 12: Anomaly Detection + Threat Intelligence

| Task | Description | Output |
|------|-------------|--------|
| 12.1 | Baseline traffic profiler (per-device normal patterns) | Baseline established |
| 12.2 | Isolation Forest anomaly detector | Anomaly scoring |
| 12.3 | Integrate anomaly scoring into L4 pipeline | Anomalies flagged |
| 12.4 | Auto-update threat feeds (daily cron, abuse.ch + OTX) | Fresh threat intel |
| 12.5 | Rogue AP detection (BSSID monitoring on WiFi networks) | Evil twin detection |
| 12.6 | Port scan detection heuristic | Scan detection |
| 12.7 | Dashboard: ML insights panel (classification confidence, anomaly scores) | ML visibility |

**Milestone:** System learns normal traffic patterns and flags genuine anomalies.

---

## Phase 6: Hardening (Week 13)

**Goal:** Production-ready, documented, demo-able.

| Task | Description | Output |
|------|-------------|--------|
| 13.1 | Integration tests (agent -> core -> db -> dashboard flow) | Test suite |
| 13.2 | Simulated attack test suite (C2, DNS tunnel, ARP spoof, etc.) | Detection validation |
| 13.3 | Performance testing (3 agents, sustained traffic) | Performance report |
| 13.4 | Docker Compose hardening (health checks, restart policies, volumes) | Production config |
| 13.5 | Setup wizard script (interactive first-time setup) | Easy onboarding |
| 13.6 | README.md with architecture diagram, setup guide, screenshots | Documentation |
| 13.7 | Demo script (reproducible demo with simulated traffic) | Portfolio demo |
| 13.8 | Security audit (no hardcoded secrets, proper auth, encrypted storage) | Security report |

**Milestone:** One-command setup, documented, demo-ready for portfolio/interviews.

---

## Dependency Graph

```
Phase 1 (Foundation)
├── 1.1-1.6: Setup + DB          ─── no deps
├── 2.1-2.6: Agent + Comms       ─── depends on 1.5 (Core running)
└── 3.1-3.7: Packet Capture      ─── depends on 2.3 (Agent connected)

Phase 2 (Pipeline)
├── 4.1-4.8: L2 + L3             ─── depends on 3.6 (packets in DB)
├── 5.1-5.8: L4 Threat Detect    ─── depends on 4.1 (flows exist)
└── 6.1-6.9: L5 Response         ─── depends on 5.1 (threats detected)

Phase 3 (Dashboard)
├── 7.1-7.7: Core pages          ─── depends on Phase 1 APIs
└── 8.1-8.10: Advanced viz       ─── depends on Phase 2 APIs

Phase 4 (CLI + Alerts)
├── 9.1-9.9: CLI                 ─── depends on Phase 1-2 APIs
└── 10.1-10.9: Telegram bot      ─── depends on 6.1 (response engine)

Phase 5 (ML)
├── 11.1-11.6: Classifier        ─── depends on 4.1 (flow data exists)
└── 12.1-12.7: Anomaly           ─── depends on 2+ weeks of flow data

Phase 6 (Hardening)
└── 13.1-13.8: All tasks         ─── depends on Phases 1-5
```

---

## Risk Mitigation Per Phase

| Phase | Risk | Mitigation |
|-------|------|------------|
| 1 | Android packet capture fails | Test Termux + tcpdump early in Week 3, have VPN-TUN ready |
| 2 | Flow assembly misses edge cases | Start with TCP only, add UDP later; test with known traffic |
| 3 | Dashboard performance with real-time data | Throttle WebSocket updates to 1Hz, paginate tables |
| 4 | Telegram bot rate limiting | Queue alerts, batch low-severity, respect Telegram limits |
| 5 | ML model accuracy too low | Always fall back to heuristic chain; ML improves, doesn't gate |
| 6 | Complex Docker setup fails on demo | Pre-build images, test on fresh machine before demo |
