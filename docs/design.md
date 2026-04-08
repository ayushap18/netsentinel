# NetSentinel — Personal Network Operations Center

**Date:** 2026-04-09
**Author:** Ayush
**Status:** Design Approved

---

## 1. Overview

NetSentinel is a distributed, multi-layered network monitoring and defense system that provides full-stack visibility (packet to application level) across all personal devices with intelligent tiered threat response.

**What makes this a technology, not a project:**
- Custom binary protocol between agents and core
- 5-layer analysis pipeline (packet -> flow -> app -> threat -> response)
- Distributed agent architecture across heterogeneous devices
- Real-time streaming via WebSockets
- Extensible threat rule engine
- ML-based traffic classification and anomaly detection
- Infrastructure-level control (iptables/nftables enforcement)

**Target devices:** 2-3 Android phones (via Termux) + laptop/PC, across WiFi and mobile data.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────┐
│                 CONTROL PLANE                     │
│  ┌───────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ Web NOC   │  │ CLI Tool │  │ Telegram Bot  │ │
│  │ Dashboard │  │ (netsen) │  │ (Alerts)      │ │
│  └─────┬─────┘  └────┬─────┘  └───────┬───────┘ │
│        └──────────────┼────────────────┘         │
│                       ▼                           │
│            ┌─────────────────────┐               │
│            │   NetSentinel Core  │               │
│            │   (FastAPI Server)  │               │
│            │  ┌───────────────┐  │               │
│            │  │ Packet Engine │  │               │
│            │  │ Flow Analyzer │  │               │
│            │  │ App Classifier│  │               │
│            │  │ Threat Engine │  │               │
│            │  │ Response Tier │  │               │
│            │  └───────────────┘  │               │
│            └──────────┬──────────┘               │
│                       │                           │
│         ┌─────────────┼─────────────┐            │
│         ▼             ▼             ▼            │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│   │TimescaleDB│ │ Redis    │ │ Rules DB │       │
│   │(packets/ │ │(realtime │ │(threat   │       │
│   │ flows)   │ │ state)   │ │ rules)   │       │
│   └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────┘
                       ▲
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Agent:   │ │ Agent:   │ │ Agent:   │
    │ Android  │ │ Android  │ │ Laptop   │
    │ Phone 1  │ │ Phone 2  │ │ (Python) │
    └──────────┘ └──────────┘ └──────────┘
```

---

## 3. The 5-Layer Analysis Pipeline

### Layer 1 — Packet Capture
- Raw packet sniffing on each device agent
- Uses Scapy (cross-platform), tcpdump (Linux/Termux), nfqueue (for interception)
- Captures: src/dst IP, src/dst port, protocol, payload size, timestamps
- Local filtering to reduce noise (ignore broadcast, mDNS, etc.)
- Sends raw + summarized data upstream to Core

### Layer 2 — Flow Analysis
- Aggregates packets into logical flows (TCP sessions, UDP streams)
- Tracks per-flow: duration, total bytes in/out, packet count, retransmissions, RTT
- Maintains a flow table in Redis for active connections
- Completed flows persist to TimescaleDB with full metadata
- Detects flow anomalies: unusually long connections, data volume spikes

### Layer 3 — App Classification
- Maps flows to applications using multiple heuristics:
  - **Port-based:** well-known ports (443=HTTPS, 53=DNS, etc.)
  - **DNS correlation:** correlate SNI/DNS queries to flow destinations
  - **DPI signatures:** identify protocols (HTTP/2, QUIC, TLS fingerprints)
  - **ML classifier:** trained on labeled traffic to classify unknown flows
- Output: each flow tagged with app name, category (social, streaming, background, system)
- Builds per-app bandwidth profiles over time

### Layer 4 — Threat Detection
- **Rule engine:** YAML-based rules for known-bad patterns
  - Known malicious IPs/domains (threat intelligence feeds)
  - DNS to suspicious TLDs
  - Connections to known C2 infrastructure
  - Unencrypted traffic carrying sensitive patterns
- **Anomaly detection (ML):**
  - Baseline normal traffic patterns per device
  - Flag deviations: unusual destination countries, off-hours traffic spikes, new persistent connections
  - Isolation Forest / Local Outlier Factor for unsupervised anomaly scoring
- **Heuristic checks:**
  - DNS tunneling detection (high-entropy subdomain queries)
  - Port scanning detection
  - ARP spoofing detection (on local WiFi)
  - Rogue access point detection (BSSID changes for same SSID)

### Layer 5 — Tiered Response

| Severity | Trigger Examples | Action |
|----------|-----------------|--------|
| **LOW** | New app making network calls, minor traffic spike | Log to DB, visible in dashboard |
| **MEDIUM** | Connection to flagged IP, unencrypted credentials, unusual DNS | Alert via Telegram + recommend action in dashboard |
| **HIGH** | Known malware C2 domain, DNS tunneling detected, data exfiltration pattern | Auto-block via iptables/nftables + alert + forensic packet snapshot saved |
| **CRITICAL** | Active MITM detected, ARP spoof, rogue AP | Quarantine device traffic + full alert + auto-save forensic dump |

Response actions are configurable per-rule via YAML policies.

---

## 4. Components

### 4.1 Device Agents

**Purpose:** Lightweight daemon running on each monitored device.

**Laptop Agent (Python):**
- Full Scapy-based packet capture
- Local flow assembly to reduce bandwidth to Core
- nfqueue integration for packet-level blocking
- Runs as a systemd service (Linux) or launchd (macOS)
- Config file for capture interfaces, filters, Core server address

**Android Agent (Python via Termux):**
- Packet capture via tcpdump (requires root) or VPN-based capture (no root, using a local TUN interface)
- Lighter processing — sends more raw data to Core for analysis
- Battery-aware: reduces capture frequency on low battery
- Connects to Core via WebSocket with automatic reconnection

**Agent-Core Protocol:**
- WebSocket-based with MessagePack serialization (compact binary, faster than JSON)
- Message types:
  - `FLOW_REPORT` — completed flow summary
  - `PACKET_FLAG` — flagged suspicious packet (full payload)
  - `HEARTBEAT` — device alive + system metrics (battery, CPU, network type)
  - `COMMAND` — Core -> Agent instructions (block IP, change capture mode)
  - `BULK_STATS` — periodic aggregated statistics
- TLS encrypted, agent authenticates with pre-shared API key

### 4.2 NetSentinel Core (FastAPI Server)

**Responsibilities:**
- Receive and process agent data streams
- Run Layer 2-5 analysis pipeline
- Store data in TimescaleDB + Redis
- Serve REST API for dashboard and CLI
- Serve WebSocket streams for real-time updates
- Execute response actions

**API Endpoints (REST):**

```
GET    /api/devices                    — list all registered devices
GET    /api/devices/{id}               — device details + status
GET    /api/devices/{id}/flows         — flows for device (filterable)
GET    /api/devices/{id}/packets       — packet captures (filterable)
GET    /api/devices/{id}/apps          — app-level bandwidth breakdown
GET    /api/stats/bandwidth            — aggregate bandwidth over time
GET    /api/stats/top-talkers          — top IPs/domains by traffic
GET    /api/threats                    — active + historical threats
GET    /api/threats/{id}               — threat detail with forensic data
POST   /api/threats/{id}/action        — manual action (block/whitelist/dismiss)
GET    /api/rules                      — list threat rules
POST   /api/rules                      — add custom rule
PUT    /api/rules/{id}                 — update rule
DELETE /api/rules/{id}                 — delete rule
GET    /api/health                     — system health
```

**WebSocket Streams:**

```
/ws/live-traffic          — real-time flow stream (all devices)
/ws/device/{id}/packets   — real-time packet stream for one device
/ws/threats               — real-time threat alerts
/ws/stats                 — live bandwidth/stats updates
```

### 4.3 Web NOC Dashboard (React + Next.js)

**Pages:**

1. **Overview** — device grid with status indicators, aggregate bandwidth gauge, threat count badges, global traffic graph (last 1h/6h/24h)
2. **Device Detail** — per-device deep dive:
   - Network info (IP, type, signal strength)
   - App bandwidth breakdown (pie chart + table)
   - Active connections list (sortable, filterable)
   - Packet inspector (click a flow to see packets)
   - Device-specific threat timeline
3. **Traffic Analysis** — cross-device traffic visualization:
   - Real-time flow graph (nodes = devices/IPs, edges = connections)
   - Top talkers (IPs, domains, apps)
   - Bandwidth over time (per device, per app, per protocol)
   - Geographic map of connection destinations (MapLibre)
4. **Threat Center** — security operations view:
   - Active threats with severity color coding
   - Threat timeline
   - One-click actions: Block, Whitelist, Investigate, Dismiss
   - Forensic detail view (captured packets for the threat event)
   - Rule management UI
5. **Settings** — device management, rule editor, notification preferences, API keys

**Real-time features:**
- All graphs update live via WebSocket
- New threat alerts slide in as toast notifications
- Device status indicators pulse on data activity

### 4.4 CLI Tool — `netsen`

Built with Python + Typer. Connects to Core's REST API.

```bash
# Device management
netsen devices                              # list all devices with status
netsen device phone-1                       # detailed device info

# Live monitoring
netsen monitor                              # live traffic overview (TUI)
netsen monitor phone-1                      # live traffic for one device
netsen monitor --filter "port:443"          # filtered live view

# Flow inspection
netsen flows phone-1                        # recent flows
netsen flows phone-1 --app instagram        # flows for specific app
netsen flows phone-1 --live                 # streaming flow output

# Packet inspection
netsen packets phone-1 --last 5m            # packets from last 5 minutes
netsen packets phone-1 --filter "dns"       # DNS packets only
netsen packets phone-1 --export capture.pcap # export to pcap

# Threats
netsen threats                              # list active threats
netsen threats --severity high              # filter by severity
netsen threats 42 --detail                  # full forensic detail

# Actions
netsen block 192.168.1.50                   # block an IP across all devices
netsen whitelist example.com                # whitelist a domain
netsen quarantine phone-1                   # isolate a device

# Stats
netsen stats bandwidth --last 24h           # bandwidth report
netsen stats top-talkers --limit 20         # top connections
netsen stats apps phone-1                   # app usage breakdown
```

### 4.5 Alert System (Telegram Bot)

**Features:**
- Tiered notifications (configurable: which severities to notify)
- Rich messages with threat details, device name, IPs involved
- Inline action buttons: [Block] [Whitelist] [Investigate] [Dismiss]
- Daily summary report: total bandwidth, top apps, threats detected, devices online
- Commands: `/status`, `/devices`, `/threats`, `/block <ip>`, `/mute 1h`

---

## 5. Data Model

### TimescaleDB Tables

```sql
-- Hypertable: packet-level data (auto-partitioned by time)
CREATE TABLE packets (
    id BIGSERIAL,
    device_id TEXT NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    src_ip INET,
    dst_ip INET,
    src_port INT,
    dst_port INT,
    protocol TEXT,         -- TCP, UDP, ICMP, etc.
    payload_size INT,
    flags TEXT,            -- TCP flags
    raw_summary JSONB,     -- additional metadata
    flow_id UUID           -- links to parent flow
);
SELECT create_hypertable('packets', 'timestamp');

-- Hypertable: flow-level data
CREATE TABLE flows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id TEXT NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    src_ip INET,
    dst_ip INET,
    src_port INT,
    dst_port INT,
    protocol TEXT,
    total_bytes_in BIGINT DEFAULT 0,
    total_bytes_out BIGINT DEFAULT 0,
    packet_count INT DEFAULT 0,
    retransmissions INT DEFAULT 0,
    avg_rtt_ms FLOAT,
    app_name TEXT,          -- classified app
    app_category TEXT,      -- social, streaming, system, etc.
    dns_name TEXT,          -- resolved domain name
    country_code TEXT,      -- GeoIP lookup
    status TEXT DEFAULT 'active'  -- active, completed, blocked
);
SELECT create_hypertable('flows', 'start_time');

-- Regular table: devices
CREATE TABLE devices (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    device_type TEXT,       -- android, laptop, pc
    last_seen TIMESTAMPTZ,
    network_type TEXT,      -- wifi, 4g, 5g
    local_ip INET,
    public_ip INET,
    status TEXT DEFAULT 'offline',  -- online, offline, quarantined
    metadata JSONB
);

-- Regular table: threats
CREATE TABLE threats (
    id SERIAL PRIMARY KEY,
    device_id TEXT REFERENCES devices(id),
    flow_id UUID,
    detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    severity TEXT NOT NULL,  -- low, medium, high, critical
    category TEXT,           -- malware_c2, dns_tunnel, arp_spoof, anomaly, etc.
    description TEXT,
    evidence JSONB,          -- captured data supporting the detection
    rule_id INT,
    status TEXT DEFAULT 'active',  -- active, blocked, dismissed, whitelisted
    action_taken TEXT,
    resolved_at TIMESTAMPTZ
);

-- Regular table: threat rules
CREATE TABLE threat_rules (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    category TEXT,
    severity TEXT NOT NULL,
    condition JSONB NOT NULL,  -- rule definition
    action TEXT NOT NULL,       -- log, alert, block, quarantine
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Redis Keys

```
device:{id}:status          — online/offline/quarantined + last heartbeat
device:{id}:flows:active    — hash of currently active flow IDs
device:{id}:bandwidth:live  — current bandwidth (bytes/sec, updated every second)
stats:global:bandwidth      — aggregate bandwidth across all devices
threats:active              — sorted set of active threat IDs by severity
```

---

## 6. Threat Rule Format (YAML)

```yaml
rules:
  - name: known_malware_c2
    description: Connection to known command-and-control server
    severity: high
    condition:
      type: ip_match
      list: threat_feeds/malware_c2_ips.txt
      direction: dst
    action: block
    notify: true

  - name: dns_tunneling
    description: High-entropy DNS subdomain queries indicating tunneling
    severity: high
    condition:
      type: dns_entropy
      threshold: 3.5
      min_query_length: 30
    action: block
    notify: true

  - name: unencrypted_credentials
    description: Plaintext HTTP with auth patterns
    severity: medium
    condition:
      type: dpi_pattern
      protocol: http
      patterns:
        - "password="
        - "Authorization: Basic"
    action: alert
    notify: true

  - name: new_external_connection
    description: Device connecting to IP not seen in last 7 days
    severity: low
    condition:
      type: anomaly_new_destination
      lookback_days: 7
    action: log
    notify: false
```

---

## 7. ML Components

### Traffic Classifier (Layer 3)
- **Model:** Random Forest / Gradient Boosted Trees (scikit-learn / XGBoost)
- **Features:** packet size distribution, inter-arrival times, port, TLS fingerprint, flow duration, bytes ratio
- **Labels:** app categories (social, streaming, gaming, productivity, system, malicious)
- **Training:** capture labeled traffic from own devices over 1-2 weeks
- **Inference:** runs on Core server, classifies flows in real-time

### Anomaly Detector (Layer 4)
- **Model:** Isolation Forest (scikit-learn)
- **Features:** hourly traffic volume, unique destination count, new IP ratio, protocol distribution, DNS query entropy
- **Training:** unsupervised on 2+ weeks of baseline traffic
- **Output:** anomaly score per time window, flagged when score exceeds threshold

---

## 8. Tech Stack Summary

| Component | Technology |
|-----------|-----------|
| Core Server | Python 3.12+, FastAPI, uvicorn, WebSockets |
| Packet Engine | Scapy, pyshark, python-nfqueue |
| Agent Protocol | WebSocket + MessagePack (msgpack) |
| Time-series DB | TimescaleDB (PostgreSQL extension) |
| Cache/Real-time | Redis |
| Dashboard | React 19, Next.js 15, TypeScript, Tailwind CSS |
| Charts | Recharts (graphs), MapLibre GL (geo map) |
| CLI | Python, Typer, Rich (for TUI) |
| Telegram Bot | python-telegram-bot |
| ML | scikit-learn, XGBoost |
| GeoIP | MaxMind GeoLite2 |
| Containerization | Docker, Docker Compose |
| Threat Feeds | abuse.ch, AlienVault OTX (free tiers) |

---

## 9. Security Considerations

- Agent-Core communication is TLS encrypted
- API keys for agent authentication (pre-shared, rotatable)
- Dashboard protected by local auth (JWT)
- Packet data retention policy: raw packets auto-deleted after 7 days, flow summaries kept 90 days
- No data leaves your local network (self-hosted only)
- CLI and Telegram bot authenticate via API key

---

## 10. Project Structure

```
netsentinel/
├── core/                       # FastAPI server
│   ├── main.py                 # App entrypoint
│   ├── config.py               # Configuration
│   ├── api/
│   │   ├── routes/
│   │   │   ├── devices.py
│   │   │   ├── flows.py
│   │   │   ├── packets.py
│   │   │   ├── threats.py
│   │   │   ├── rules.py
│   │   │   └── stats.py
│   │   └── websockets/
│   │       ├── live_traffic.py
│   │       └── threat_stream.py
│   ├── pipeline/
│   │   ├── packet_engine.py    # L1: packet processing
│   │   ├── flow_analyzer.py    # L2: flow assembly
│   │   ├── app_classifier.py   # L3: app identification
│   │   ├── threat_detector.py  # L4: threat detection
│   │   └── response_engine.py  # L5: tiered response
│   ├── models/
│   │   ├── database.py         # SQLAlchemy models
│   │   └── schemas.py          # Pydantic schemas
│   ├── services/
│   │   ├── agent_manager.py    # Manages agent connections
│   │   ├── rule_engine.py      # Threat rule evaluation
│   │   └── alert_service.py    # Telegram/Discord notifications
│   └── ml/
│       ├── traffic_classifier.py
│       └── anomaly_detector.py
├── agent/                      # Device agent
│   ├── main.py
│   ├── capture/
│   │   ├── scapy_capture.py    # Laptop capture
│   │   ├── tcpdump_capture.py  # Android capture
│   │   └── vpn_capture.py      # No-root Android capture
│   ├── processor.py            # Local flow assembly
│   ├── transport.py            # WebSocket client to Core
│   └── enforcer.py             # iptables/nftables block execution
├── dashboard/                  # Next.js frontend
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx        # Overview
│   │   │   ├── devices/
│   │   │   ├── traffic/
│   │   │   ├── threats/
│   │   │   └── settings/
│   │   ├── components/
│   │   │   ├── DeviceGrid.tsx
│   │   │   ├── FlowTable.tsx
│   │   │   ├── PacketInspector.tsx
│   │   │   ├── ThreatCard.tsx
│   │   │   ├── BandwidthChart.tsx
│   │   │   ├── GeoMap.tsx
│   │   │   └── LiveIndicator.tsx
│   │   ├── hooks/
│   │   │   ├── useWebSocket.ts
│   │   │   └── useDeviceData.ts
│   │   └── lib/
│   │       └── api.ts
│   └── package.json
├── cli/                        # CLI tool
│   ├── main.py
│   ├── commands/
│   │   ├── devices.py
│   │   ├── monitor.py
│   │   ├── flows.py
│   │   ├── packets.py
│   │   ├── threats.py
│   │   └── stats.py
│   └── display.py              # Rich TUI formatting
├── bot/                        # Telegram bot
│   ├── main.py
│   ├── handlers.py
│   └── formatters.py
├── rules/                      # Threat rules (YAML)
│   ├── default_rules.yaml
│   └── custom_rules.yaml
├── threat_feeds/               # Downloaded threat intel
│   └── README.md
├── ml_models/                  # Trained model artifacts
│   └── README.md
├── docker-compose.yml
├── Dockerfile.core
├── Dockerfile.agent
├── Dockerfile.dashboard
├── pyproject.toml
└── README.md
```

---

## 11. Success Criteria

1. **Agents connect reliably** from Android (Termux) and laptop to Core server
2. **Packet capture works** on both device types with <5% packet loss
3. **5-layer pipeline processes** flows in real-time (<2s from packet to dashboard)
4. **Dashboard shows** live traffic, per-app breakdown, and drill-down to packet level
5. **Threat detection fires** correctly on test scenarios (simulated C2, DNS tunnel, ARP spoof)
6. **Tiered response executes** — low logs, medium alerts, high blocks automatically
7. **CLI provides** full inspection capability matching dashboard functionality
8. **Telegram alerts** arrive within 5 seconds of threat detection
9. **System handles** 3+ simultaneous device agents without degradation

---

## 12. Non-Goals (Explicit Exclusions)

- No cloud deployment — this runs entirely on your local network
- No iOS agent — Termux/Python approach works on Android only
- No commercial threat intelligence (paid feeds) — free feeds only
- No VPN/proxy functionality — this monitors, it doesn't route
- No web-based packet replay — forensic data viewable but not replayable in browser
