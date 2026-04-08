<div align="center">

```
                    _   _      _   ____             _   _            _ 
                   | \ | | ___| |_/ ___|  ___ _ __ | |_(_)_ __   ___| |
                   |  \| |/ _ \ __\___ \ / _ \ '_ \| __| | '_ \ / _ \ |
                   | |\  |  __/ |_ ___) |  __/ | | | |_| | | | |  __/ |
                   |_| \_|\___|\__|____/ \___|_| |_|\__|_|_| |_|\___|_|
```

### Your Network. Your Rules. Your Eyes Everywhere.

[![Python](https://img.shields.io/badge/Python-3.12+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Next.js](https://img.shields.io/badge/Next.js_15-000000?style=for-the-badge&logo=nextdotjs&logoColor=white)](https://nextjs.org)
[![React](https://img.shields.io/badge/React_19-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://typescriptlang.org)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![PostgreSQL](https://img.shields.io/badge/TimescaleDB-FDB515?style=for-the-badge&logo=timescale&logoColor=black)](https://timescale.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

**A personal Network Operations Center that sees every packet, classifies every app,
detects every threat, and defends every device you own — in real time.**

[Get Started](#-quick-start) | [Architecture](#-how-it-works) | [Documentation](#-documentation) | [Demo](#-see-it-in-action)

</div>

---

<br>

## The Problem

You have multiple devices. They're constantly talking to the internet. **You have no idea what they're saying.**

- Which IPs is your phone secretly connecting to at 3 AM?
- Is that free WiFi at the cafe intercepting your traffic?
- Which app is burning through your mobile data?
- Is something on your laptop phoning home to a C2 server?

Your phone's built-in data monitor shows you **megabytes**. NetSentinel shows you **everything**.

<br>

## The Solution

```
              YOU OWN THE ENTIRE STACK
              
    [Your Devices]          [Your Server]           [Your Eyes]
         |                       |                       |
    Packet Capture          5-Layer Deep             Dashboard
    on every device    -->  Analysis Pipeline   -->  CLI Tool
    you own                 running on your          Telegram Alerts
                            machine                  on your phone
```

<br>

## What Makes This Different

> This is not another network monitoring **app**.
> This is a network monitoring **technology**.

| Most "Network Monitors" | NetSentinel |
|:-----------------------:|:-----------:|
| Show total data usage | Captures every packet |
| Run on one device | Distributed agents across all your devices |
| Pre-built dashboard | Custom 5-layer analysis pipeline |
| Alert on high usage | ML-powered anomaly detection + auto-blocking |
| REST API polling | Real-time WebSocket streaming (<2s latency) |
| Use existing protocols | Custom binary protocol (MessagePack over WSS) |

<br>

---

## How It Works

<br>

### The Pipeline: 5 Layers Deep

Every packet that enters or leaves your device passes through five increasingly intelligent layers:

```
RAW PACKET
    |
    v
[ L1: PACKET ENGINE ]     Capture raw packets via Scapy/tcpdump
    |                      Extract: IPs, ports, protocol, flags, payload size
    v
[ L2: FLOW ANALYZER ]     Assemble packets into TCP sessions & UDP flows
    |                      Track: duration, bytes, RTT, retransmissions
    v
[ L3: APP CLASSIFIER ]    Identify the application behind each flow
    |                      Methods: port mapping + DNS + TLS SNI + DPI + ML
    v
[ L4: THREAT DETECTOR ]   Evaluate against threat rules + ML anomaly model
    |                      Detect: C2, DNS tunneling, ARP spoof, data exfil
    v
[ L5: RESPONSE ENGINE ]   Execute tiered action based on severity
                           LOW    --> log silently
                           MEDIUM --> alert via Telegram + recommend
                           HIGH   --> auto-block + alert + save forensics
                           CRITICAL -> quarantine device + full alert
```

<br>

### The Architecture

```
                         +------------------------------------------+
                         |            CONTROL PLANE                  |
                         |                                          |
                         |  +----------+  +------+  +------------+ |
                         |  | Web NOC  |  | CLI  |  | Telegram   | |
                         |  | Dashboard|  | Tool |  | Bot        | |
                         |  +----+-----+  +--+---+  +-----+------+ |
                         |       |           |             |        |
                         |       +-----+-----+-------------+        |
                         |             |                            |
                         |       REST API + WebSocket               |
                         +-------------+----------------------------+
                                       |
                         +-------------v----------------------------+
                         |         NETSENTINEL CORE                 |
                         |                                          |
                         |    Agent Manager (WebSocket Server)      |
                         |              |                           |
                         |    +---------v----------+                |
                         |    | Analysis Pipeline  |                |
                         |    | L1 > L2 > L3 > L4 > L5             |
                         |    +--------+-----------+                |
                         |             |                            |
                         |    +--------v---------+  +-----------+  |
                         |    | TimescaleDB      |  | Redis     |  |
                         |    | (packets, flows) |  | (realtime)|  |
                         |    +---------+--------+  +-----------+  |
                         +-------------+----------------------------+
                                       ^
                                       | WebSocket + MessagePack
                          +------------+-------------+
                          |            |             |
                    +-----+----+ +----+-----+ +-----+----+
                    |  Agent   | |  Agent   | |  Agent   |
                    |  Laptop  | |  Phone 1 | |  Phone 2 |
                    |  (Scapy) | | (Termux) | | (Termux) |
                    +----------+ +----------+ +----------+
```

<br>

---

## See It In Action

### Web NOC Dashboard
```
+------------------------------------------------------------------+
|  NETSENTINEL                          [3 devices online] [2 threats]
|------------------------------------------------------------------|
|                                                                    |
|  +------------------+  +------------------+  +------------------+ |
|  | Ayush's Phone    |  | Ayush's Laptop   |  | Phone 2          | |
|  | ONLINE    4G     |  | ONLINE    WiFi   |  | OFFLINE          | |
|  | 2.4 MB/s  12 apps|  | 5.1 MB/s  8 apps |  | Last: 2h ago     | |
|  +------------------+  +------------------+  +------------------+ |
|                                                                    |
|  Bandwidth (Last 1 Hour)                                          |
|  8 MB/s |    __                                                    |
|  6 MB/s |   /  \    __                                             |
|  4 MB/s |  /    \__/  \         __                                 |
|  2 MB/s | /            \___/\__/  \_____                           |
|  0 MB/s +-----------------------------------------> time           |
|                                                                    |
|  Active Threats                                                    |
|  [!] HIGH   DNS tunneling detected on Phone 1        [Block]      |
|  [i] LOW    New destination: 203.0.113.50 (Russia)   [Inspect]    |
+------------------------------------------------------------------+
```

### CLI Tool
```bash
$ netsen monitor

  NETSENTINEL LIVE MONITOR                          3 devices | 2 threats
  ================================================================

  DEVICE          STATUS    BANDWIDTH     FLOWS    NETWORK
  ayush-phone     ONLINE    2.4 MB/s      12       4G
  ayush-laptop    ONLINE    5.1 MB/s      8        WiFi
  phone-2         OFFLINE   ---           ---      ---

  TOP APPS (ayush-phone)
  Instagram       45%   =============================
  Chrome          22%   ==============
  WhatsApp        15%   ==========
  YouTube          8%   =====
  System          10%   ======

  LIVE FLOWS
  10:30:01  phone -> 157.240.1.35:443   instagram   TLS 1.3   1.2 MB
  10:30:02  laptop -> 142.250.80.46:443  google      TLS 1.3   340 KB
  10:30:03  phone -> 31.13.72.36:443     whatsapp    TLS 1.3   12 KB
```

### Telegram Alerts
```
+--------------------------------------------+
|  NETSENTINEL ALERT                         |
|                                            |
|  Severity: HIGH                            |
|  Device:   Ayush's Phone                   |
|  Category: DNS Tunneling                   |
|                                            |
|  Suspicious high-entropy DNS queries       |
|  detected to d3adb33f.evil.com             |
|  (entropy: 4.2, threshold: 3.5)           |
|                                            |
|  Action taken: AUTO-BLOCKED                |
|  Forensic dump: saved                      |
|                                            |
|  [Block IP]  [Whitelist]  [Investigate]    |
+--------------------------------------------+
```

<br>

---

## Quick Start

### 1. Clone & Configure
```bash
git clone https://github.com/ayushap18/netsentinel.git
cd netsentinel

cp .env.example .env
# Edit .env -> set passwords, Telegram token, etc.
```

### 2. Launch Core (One Command)
```bash
docker-compose up -d

# Verify
curl http://localhost:8000/api/health
```

### 3. Setup Admin
```bash
docker exec -it netsentinel-core python -m netsentinel.setup
```

### 4. Connect Your Laptop
```bash
cd agent && pip install -e .

netsen-agent setup --server wss://localhost:8000/ws/agent --name "my-laptop"
netsen-agent start
```

### 5. Connect Your Phone (Termux)
```bash
# In Termux on Android:
pip install netsentinel-agent

netsen-agent setup \
    --server wss://YOUR_LAPTOP_IP:8000/ws/agent \
    --name "my-phone" \
    --capture-mode tcpdump   # or vpn-tun (no root)

netsen-agent start
```

### 6. Open Dashboard
```
http://localhost:3000
```

> **Detailed setup:** [Setup Guide](docs/guides/SETUP-GUIDE.md) | [Android Guide](docs/guides/ANDROID-AGENT-GUIDE.md)

<br>

---

## Tech Stack

```
                    +------------------+
                    |    FRONTEND      |
                    |                  |
                    |  Next.js 15      |
                    |  React 19        |
                    |  TypeScript      |
                    |  Tailwind CSS    |
                    |  Recharts        |
                    |  MapLibre GL     |
                    +--------+---------+
                             |
                    +--------v---------+
                    |    BACKEND       |
                    |                  |
                    |  FastAPI         |
                    |  WebSockets      |
                    |  Scapy           |
                    |  scikit-learn    |
                    |  MessagePack     |
                    +--------+---------+
                             |
                    +--------v---------+
                    |    DATA          |
                    |                  |
                    |  TimescaleDB     |
                    |  Redis           |
                    |  Alembic         |
                    +--------+---------+
                             |
                    +--------v---------+
                    |    INFRA         |
                    |                  |
                    |  Docker Compose  |
                    |  Telegram Bot    |
                    |  Typer + Rich    |
                    +------------------+
```

<br>

---

## Project Structure

```
netsentinel/
|
+-- core/                          # The brain
|   +-- api/                       #   REST + WebSocket endpoints
|   +-- pipeline/                  #   5-layer analysis engine
|   |   +-- packet_engine.py       #     L1: raw packet processing
|   |   +-- flow_analyzer.py       #     L2: TCP/UDP flow assembly
|   |   +-- app_classifier.py      #     L3: app identification (DPI + ML)
|   |   +-- threat_detector.py     #     L4: rule engine + anomaly detection
|   |   +-- response_engine.py     #     L5: tiered automated response
|   +-- services/                  #   agent mgmt, alerts, rules
|   +-- ml/                        #   traffic classifier, anomaly detector
|   +-- models/                    #   SQLAlchemy + Pydantic
|
+-- agent/                         # The eyes (runs on each device)
|   +-- capture/                   #   Scapy, tcpdump, VPN-TUN
|   +-- processor.py               #   local flow assembly
|   +-- transport.py               #   WebSocket + MessagePack client
|   +-- enforcer.py                #   iptables/nftables execution
|
+-- dashboard/                     # The face (Next.js)
|   +-- src/app/                   #   pages: overview, devices, traffic, threats
|   +-- src/components/            #   DeviceGrid, FlowTable, GeoMap, ThreatCard
|   +-- src/hooks/                 #   useWebSocket, useDeviceData
|
+-- cli/                           # The terminal (netsen command)
|   +-- commands/                  #   devices, monitor, flows, packets, threats
|   +-- display.py                 #   Rich TUI formatting
|
+-- bot/                           # The messenger (Telegram)
+-- rules/                         # Threat detection rules (YAML)
+-- docs/                          # You are here
+-- docker-compose.yml             # One command to rule them all
```

<br>

---

## Skills Demonstrated

This project isn't just code. It's proof of **systems-level thinking**.

```
+----------------------+------------------------------------------------+
| Skill                | How NetSentinel Proves It                      |
+----------------------+------------------------------------------------+
| Distributed Systems  | Multi-device agents coordinated by central     |
|                      | server via custom binary protocol               |
+----------------------+------------------------------------------------+
| Network Security     | IDS/IPS with rule engine, threat intel feeds,  |
|                      | automated blocking, forensic evidence capture   |
+----------------------+------------------------------------------------+
| Protocol Design      | Custom MessagePack-over-WebSocket protocol      |
|                      | with auth, heartbeats, and bidirectional cmds   |
+----------------------+------------------------------------------------+
| Data Engineering     | 5-stage pipeline processing packets through     |
|                      | increasingly abstract analysis layers           |
+----------------------+------------------------------------------------+
| Machine Learning     | Random Forest traffic classifier + Isolation    |
|                      | Forest anomaly detector on real network data    |
+----------------------+------------------------------------------------+
| Full-Stack Dev       | FastAPI backend + Next.js dashboard + CLI +     |
|                      | Telegram bot + Docker deployment                |
+----------------------+------------------------------------------------+
| Database Design      | TimescaleDB hypertables with auto-partitioning, |
|                      | retention, compression + Redis real-time state  |
+----------------------+------------------------------------------------+
| Infrastructure       | Agent-level traffic enforcement via iptables,   |
|                      | device quarantine, automated threat response    |
+----------------------+------------------------------------------------+
```

<br>

---

## Documentation

| | Document | What You'll Learn |
|---|----------|------------------|
| **Blueprint** | [PRD](docs/PRD.md) | Requirements, user stories, success metrics |
| **Brain** | [Architecture](docs/architecture/ARCHITECTURE.md) | System design, data flow, component interaction |
| **Data** | [Data Model](docs/architecture/DATA-MODEL.md) | Schema, Redis keys, retention, migrations |
| **Interface** | [API Reference](docs/architecture/API-REFERENCE.md) | Every REST + WebSocket endpoint |
| **Shield** | [Security](docs/architecture/SECURITY.md) | Threat model, auth, encryption, hardening |
| **Roadmap** | [Implementation Plan](docs/plans/IMPLEMENTATION-PLAN.md) | 6-phase weekly build plan |
| **Decisions** | [Tech Decisions](docs/plans/TECH-DECISIONS.md) | Why each tech was chosen (with alternatives) |
| **Milestones** | [Milestones](docs/plans/MILESTONES.md) | 8 demo checkpoints + what to say in interviews |
| **Setup** | [Setup Guide](docs/guides/SETUP-GUIDE.md) | Full installation walkthrough |
| **Mobile** | [Android Guide](docs/guides/ANDROID-AGENT-GUIDE.md) | Termux agent on Android |
| **Origin** | [Design Spec](docs/design.md) | Original design document |

<br>

---

## Threat Response in Action

```
Normal traffic:       You --> instagram.com       [ L4: CLEAN ]     --> log quietly

Suspicious DNS:       You --> x8f2k.evil.com      [ L4: MEDIUM ]    --> Telegram alert
                                                                        "Block this?"

Known C2 server:      You --> 198.51.100.50:4444  [ L4: HIGH ]      --> AUTO-BLOCKED
                                                                        forensics saved
                                                                        Telegram: "Blocked."

Active MITM attack:   Attacker --> ARP spoof      [ L4: CRITICAL ]  --> QUARANTINED
                                                                        all traffic halted
                                                                        full dump saved
                                                                        URGENT alert sent
```

<br>

---

## Roadmap

- [x] System design & architecture
- [x] Complete documentation suite
- [ ] Phase 1: Core server + agent + packet capture
- [ ] Phase 2: 5-layer analysis pipeline
- [ ] Phase 3: Web NOC dashboard
- [ ] Phase 4: CLI tool + Telegram alerts
- [ ] Phase 5: ML traffic classifier + anomaly detection
- [ ] Phase 6: Hardening + demo preparation

<br>

---

<div align="center">

### Built by Ayush

**Cybersecurity Enthusiast | Full-Stack Developer | Builder**

If this project interests you, star it and watch the build journey.

Every packet tells a story. NetSentinel reads them all.

</div>
