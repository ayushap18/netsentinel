# NetSentinel — System Architecture

**Version:** 1.0
**Date:** 2026-04-09

---

## 1. High-Level Architecture

NetSentinel follows a **hub-and-spoke distributed architecture** where lightweight agents on each device (spokes) stream data to a central Core server (hub) that runs the analysis pipeline and serves the control interfaces.

```
                    ┌─────────────────────────┐
                    │      CONTROL PLANE       │
                    │                          │
                    │  Dashboard  CLI   Bot    │
                    │     ▲        ▲     ▲     │
                    │     │        │     │     │
                    │     └────┬───┘─────┘     │
                    │          │               │
                    │    REST API + WebSocket   │
                    └──────────┬───────────────┘
                               │
                    ┌──────────▼───────────────┐
                    │     NETSENTINEL CORE      │
                    │                           │
                    │  ┌─────────────────────┐  │
                    │  │  Agent Manager      │  │
                    │  │  (WebSocket Server) │  │
                    │  └────────┬────────────┘  │
                    │           │               │
                    │  ┌────────▼────────────┐  │
                    │  │  Analysis Pipeline  │  │
                    │  │                     │  │
                    │  │  L1: Packet Engine  │  │
                    │  │  L2: Flow Analyzer  │  │
                    │  │  L3: App Classifier │  │
                    │  │  L4: Threat Detect  │  │
                    │  │  L5: Response Tier  │  │
                    │  └────────┬────────────┘  │
                    │           │               │
                    │  ┌────────▼────────────┐  │
                    │  │  Storage Layer      │  │
                    │  │  TimescaleDB Redis  │  │
                    │  └─────────────────────┘  │
                    └──────────▲───────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
     │  Agent:       │ │  Agent:      │ │  Agent:      │
     │  Laptop       │ │  Android #1  │ │  Android #2  │
     │  (Scapy)      │ │  (Termux)    │ │  (Termux)    │
     └───────────────┘ └──────────────┘ └──────────────┘
```

---

## 2. Component Architecture

### 2.1 Device Agent

The agent is the data collection layer. It runs on each monitored device and is responsible for packet capture, local pre-processing, and communication with Core.

```
┌─────────────────────────────────────────┐
│              DEVICE AGENT               │
│                                         │
│  ┌─────────────┐    ┌───────────────┐  │
│  │   Capture    │    │   Enforcer    │  │
│  │   Module     │    │   Module      │  │
│  │             │    │               │  │
│  │  Scapy     │    │  iptables/    │  │
│  │  tcpdump   │    │  nftables     │  │
│  │  VPN-TUN   │    │  block/allow  │  │
│  └──────┬──────┘    └───────▲───────┘  │
│         │                    │          │
│  ┌──────▼──────────────────────────┐   │
│  │       Local Processor           │   │
│  │                                 │   │
│  │  - Packet filtering             │   │
│  │  - Flow assembly (basic)        │   │
│  │  - Noise reduction              │   │
│  │  - Metric aggregation           │   │
│  └──────┬──────────────────────────┘   │
│         │                               │
│  ┌──────▼──────────────────────────┐   │
│  │       Transport Layer           │   │
│  │                                 │   │
│  │  WebSocket + MessagePack        │   │
│  │  TLS encrypted                  │   │
│  │  Auto-reconnect                 │   │
│  │  Heartbeat (30s interval)       │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Platform-specific capture strategies:**

| Platform | Primary Method | Fallback | Root Required |
|----------|---------------|----------|---------------|
| Linux laptop | Scapy (raw sockets) | tcpdump subprocess | Yes (or CAP_NET_RAW) |
| macOS laptop | Scapy (BPF) | tcpdump subprocess | Yes (sudo) |
| Android (rooted) | tcpdump via Termux | — | Yes |
| Android (no root) | VPN-TUN interface | — | No |

### 2.2 Core Server

The brain of NetSentinel. Receives agent data, runs the analysis pipeline, stores results, and serves all control interfaces.

```
┌──────────────────────────────────────────────────────┐
│                   CORE SERVER (FastAPI)               │
│                                                       │
│  ┌──────────────────┐     ┌────────────────────────┐ │
│  │  Agent Manager   │     │    API Layer           │ │
│  │                  │     │                        │ │
│  │  - WS handler    │     │  REST: /api/*          │ │
│  │  - Auth          │     │  WS: /ws/*             │ │
│  │  - Session mgmt  │     │  JWT auth              │ │
│  │  - Command dispatch│   │  Rate limiting         │ │
│  └────────┬─────────┘     └────────────────────────┘ │
│           │                                           │
│  ┌────────▼──────────────────────────────────────┐   │
│  │            ANALYSIS PIPELINE                   │   │
│  │                                                │   │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │   │
│  │  │ L1       │  │ L2       │  │ L3          │ │   │
│  │  │ Packet   │─▶│ Flow     │─▶│ App         │ │   │
│  │  │ Engine   │  │ Analyzer │  │ Classifier  │ │   │
│  │  └──────────┘  └──────────┘  └──────┬──────┘ │   │
│  │                                      │        │   │
│  │  ┌──────────┐  ┌──────────┐         │        │   │
│  │  │ L5       │  │ L4       │◀────────┘        │   │
│  │  │ Response │◀─│ Threat   │                   │   │
│  │  │ Engine   │  │ Detector │                   │   │
│  │  └──────────┘  └──────────┘                   │   │
│  └───────────────────────────────────────────────┘   │
│                                                       │
│  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  Alert Service   │  │  Rule Engine             │  │
│  │                  │  │                          │  │
│  │  - Telegram bot  │  │  - YAML rule loader      │  │
│  │  - Discord       │  │  - Condition evaluator   │  │
│  │  - Daily digest  │  │  - Threat feed updater   │  │
│  └──────────────────┘  └──────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

### 2.3 Data Layer

```
┌─────────────────────────────────────────────────────┐
│                    DATA LAYER                        │
│                                                      │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │    TimescaleDB      │  │       Redis          │  │
│  │    (Persistent)     │  │    (Real-time)       │  │
│  │                     │  │                      │  │
│  │  - packets (hyper)  │  │  - device status     │  │
│  │  - flows (hyper)    │  │  - active flows      │  │
│  │  - devices          │  │  - live bandwidth    │  │
│  │  - threats          │  │  - active threats    │  │
│  │  - threat_rules     │  │  - pub/sub channels  │  │
│  │                     │  │                      │  │
│  │  Retention:         │  │  TTL-based expiry    │  │
│  │  packets: 7 days    │  │                      │  │
│  │  flows: 90 days     │  │                      │  │
│  │  threats: forever   │  │                      │  │
│  └─────────────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 3. Data Flow

### 3.1 Packet Ingestion Flow

```
Agent captures packet
        │
        ▼
Local filter (drop noise: mDNS, broadcast, ARP broadcast)
        │
        ▼
Local flow assembly (group packets into flows)
        │
        ▼
Serialize with MessagePack
        │
        ▼
Send via WebSocket (TLS) ──────────────▶ Core: Agent Manager
                                                  │
                                                  ▼
                                         Deserialize + validate
                                                  │
                                                  ▼
                                         L1: Extract packet metadata
                                                  │
                                                  ▼
                                         L2: Update flow table (Redis)
                                                  │
                                                  ▼
                                         L3: Classify app
                                                  │
                                                  ▼
                                         L4: Evaluate threat rules + ML
                                                  │
                                         ┌────────┴────────┐
                                         │                  │
                                    No threat           Threat found
                                         │                  │
                                         ▼                  ▼
                                    Store flow         L5: Tiered response
                                    in TimescaleDB     (log/alert/block/quarantine)
                                         │                  │
                                         ▼                  ▼
                                    Push to WebSocket   Alert + Store + Action
                                    (dashboard/CLI)     (Telegram + DB + iptables)
```

### 3.2 Threat Response Flow

```
Threat detected by L4
        │
        ▼
Determine severity (LOW / MEDIUM / HIGH / CRITICAL)
        │
        ├─── LOW ──────▶ Log to TimescaleDB
        │                Update dashboard via WebSocket
        │
        ├─── MEDIUM ───▶ Log to TimescaleDB
        │                Alert via Telegram (with recommendation)
        │                Update dashboard + highlight
        │
        ├─── HIGH ─────▶ Log to TimescaleDB
        │                Save forensic packet snapshot
        │                Send BLOCK command to agent
        │                Alert via Telegram (with "Blocked" status)
        │                Update dashboard + highlight
        │
        └─── CRITICAL ─▶ Log to TimescaleDB
                         Save full forensic dump
                         Send QUARANTINE command to agent
                         Alert via Telegram (urgent)
                         Update dashboard + highlight + alarm
```

---

## 4. Communication Protocols

### 4.1 Agent ↔ Core Protocol

**Transport:** WebSocket over TLS (wss://)
**Serialization:** MessagePack (binary, ~30% smaller than JSON)
**Authentication:** API key in WebSocket handshake header

**Message Envelope:**
```
{
    "type": "FLOW_REPORT" | "PACKET_FLAG" | "HEARTBEAT" | "BULK_STATS" | "COMMAND" | "ACK",
    "device_id": "phone-1",
    "timestamp": 1712678400.123,
    "seq": 42,
    "payload": { ... }
}
```

**Message Types:**

| Type | Direction | Purpose | Frequency |
|------|-----------|---------|-----------|
| `HEARTBEAT` | Agent → Core | Device alive + metrics | Every 30s |
| `FLOW_REPORT` | Agent → Core | Completed flow summary | Per flow completion |
| `PACKET_FLAG` | Agent → Core | Suspicious packet (full data) | On detection |
| `BULK_STATS` | Agent → Core | Aggregated stats batch | Every 60s |
| `COMMAND` | Core → Agent | Block/allow/config change | On demand |
| `ACK` | Both | Acknowledge critical messages | Per critical msg |

### 4.2 Core ↔ Dashboard Protocol

**REST API:** Standard HTTP/JSON for queries and actions
**WebSocket:** For real-time streaming updates

**WebSocket Channels:**
```
/ws/live-traffic          — all device traffic (1 msg/sec aggregate)
/ws/device/{id}/packets   — per-device packet stream
/ws/threats               — new threat alerts
/ws/stats                 — bandwidth + device stats
```

---

## 5. Security Architecture

```
┌───────────────────────────────────────────────┐
│              SECURITY BOUNDARIES               │
│                                                │
│   Agent ◄──── TLS + API Key ────► Core        │
│                                                │
│   Dashboard ◄── TLS + JWT ──────► Core API    │
│                                                │
│   CLI ◄──────── API Key ────────► Core API    │
│                                                │
│   Telegram ◄─── Bot Token ──────► Bot Server  │
│                                                │
│   ┌─────────────────────────────────────┐     │
│   │         DATA AT REST                │     │
│   │                                     │     │
│   │  TimescaleDB: encrypted volume      │     │
│   │  API keys: hashed (bcrypt)          │     │
│   │  Forensic dumps: encrypted files    │     │
│   └─────────────────────────────────────┘     │
└───────────────────────────────────────────────┘
```

**Key security decisions:**
1. **No data leaves local network** — everything self-hosted
2. **Agent keys are pre-shared** — generated during setup, stored in config
3. **JWT for dashboard** — short-lived tokens (1h), refresh token rotation
4. **Forensic data encrypted** — AES-256 encryption for saved packet dumps
5. **No default passwords** — setup wizard generates random credentials

---

## 6. Deployment Architecture

```
docker-compose.yml
├── core         (FastAPI server)        :8000
├── dashboard    (Next.js)               :3000
├── timescaledb  (PostgreSQL + Timescale):5432
├── redis        (Cache + pub/sub)       :6379
└── bot          (Telegram bot)          :8001

Agents run OUTSIDE Docker (directly on devices)
```

**Single-command setup:**
```bash
git clone https://github.com/ayush/netsentinel.git
cd netsentinel
cp .env.example .env        # Edit with your Telegram token, etc.
docker-compose up -d        # Starts Core + Dashboard + DBs + Bot
```

**Agent setup:**
```bash
# On laptop:
pip install netsentinel-agent
netsen-agent setup --server wss://192.168.1.100:8000 --name "laptop"
netsen-agent start

# On Android (Termux):
pkg install python
pip install netsentinel-agent
netsen-agent setup --server wss://192.168.1.100:8000 --name "phone-1"
netsen-agent start
```

---

## 7. Technology Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Core framework | FastAPI | Async-first, WebSocket support, Pydantic validation, Ayush's primary stack |
| Agent language | Python | Cross-platform (laptop + Termux), Scapy integration |
| Agent-Core serialization | MessagePack | Binary format, 30% smaller than JSON, fast encode/decode |
| Time-series DB | TimescaleDB | PostgreSQL-compatible, hypertables for packet/flow data, built-in retention policies |
| Real-time cache | Redis | Pub/sub for WebSocket fan-out, fast device state lookups |
| Dashboard | Next.js + React | SSR for initial load, React for interactive real-time UI |
| Charts | Recharts | React-native charting, supports real-time updates |
| Map | MapLibre GL | Open-source, no API key needed, vector tiles |
| CLI framework | Typer + Rich | Type-safe CLI with beautiful TUI output |
| ML framework | scikit-learn | Lightweight, sufficient for classification/anomaly tasks |
| Containerization | Docker Compose | Single-command deployment of all services |
| Alert delivery | Telegram Bot API | Free, reliable, inline buttons for actions |

---

## 8. Scalability Considerations

**Current target:** 2-5 devices, single Core server.

**If scaling needed later:**
- Core is stateless (except Redis/DB connections) — can run multiple instances behind a load balancer
- TimescaleDB supports distributed hypertables for multi-node
- Redis can be replaced with Redis Cluster
- Agent protocol is device-agnostic — new device types just need a capture module
- Rule engine is YAML-based — rules can be version-controlled and deployed independently
