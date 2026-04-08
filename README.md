# NetSentinel

**Personal Network Operations Center — Full-Stack Network Monitoring & Defense Technology**

A distributed, multi-layered network monitoring and defense system that provides packet-to-application visibility across all your personal devices with intelligent, tiered threat response.

---

## What Is This?

NetSentinel is not a simple network monitor. It's a **complete technology stack** that brings enterprise-grade Network Operations Center (NOC) capabilities to your personal devices.

```
Your Devices                    NetSentinel Core                 You
┌──────────┐                   ┌─────────────────┐            ┌──────────────┐
│ Phone 1  │──agent──┐         │ 5-Layer Pipeline │            │ Web Dashboard│
│ Phone 2  │──agent──┼──WSS──▶│ Packet → Flow →  │──API────▶│ CLI Tool     │
│ Laptop   │──agent──┘         │ App → Threat →   │            │ Telegram Bot │
└──────────┘                   │ Response         │            └──────────────┘
                               └─────────────────┘
```

## Key Features

- **5-Layer Analysis Pipeline** — Packet capture → Flow assembly → App classification → Threat detection → Automated response
- **Distributed Device Agents** — Lightweight agents on Android (Termux) and laptop capture traffic and stream to central Core
- **Real-Time Web Dashboard** — Live traffic graphs, device drill-down, geographic map, threat center
- **ML-Powered Intelligence** — Traffic classification (Random Forest) and anomaly detection (Isolation Forest)
- **Tiered Threat Response** — Low: log | Medium: alert | High: auto-block | Critical: quarantine
- **CLI Power Tool** — Full terminal-based monitoring with live TUI
- **Telegram Alerts** — Instant notifications with actionable buttons

## Architecture

| Component | Technology |
|-----------|-----------|
| Core Server | Python, FastAPI, WebSockets |
| Packet Engine | Scapy, tcpdump, nfqueue |
| Database | TimescaleDB (time-series), Redis (real-time) |
| Dashboard | React 19, Next.js 15, TypeScript, Tailwind CSS |
| CLI | Python, Typer, Rich |
| ML | scikit-learn, XGBoost |
| Alerts | Telegram Bot API |
| Deploy | Docker Compose |

## Quick Start

```bash
# Clone
git clone https://github.com/ayush/netsentinel.git
cd netsentinel

# Configure
cp .env.example .env
# Edit .env with your passwords and Telegram token

# Start Core services
docker-compose up -d

# Run setup wizard
docker exec -it netsentinel-core python -m netsentinel.setup

# Install laptop agent
cd agent && pip install -e .
netsen-agent setup --server wss://localhost:8000/ws/agent --name "my-laptop"
netsen-agent start

# Open dashboard
open http://localhost:3000
```

See [Setup Guide](docs/guides/SETUP-GUIDE.md) for detailed instructions including Android setup.

## Documentation

| Document | Description |
|----------|-------------|
| [PRD](docs/PRD.md) | Product Requirements Document — what and why |
| [Architecture](docs/architecture/ARCHITECTURE.md) | System architecture, components, data flow |
| [Data Model](docs/architecture/DATA-MODEL.md) | Database schema, Redis keys, retention policies |
| [API Reference](docs/architecture/API-REFERENCE.md) | REST + WebSocket API documentation |
| [Security](docs/architecture/SECURITY.md) | Security architecture, threat model, encryption |
| [Implementation Plan](docs/plans/IMPLEMENTATION-PLAN.md) | Phased build plan with weekly tasks |
| [Tech Decisions](docs/plans/TECH-DECISIONS.md) | Why each technology was chosen |
| [Milestones](docs/plans/MILESTONES.md) | Demo-able checkpoints with portfolio talking points |
| [Setup Guide](docs/guides/SETUP-GUIDE.md) | Complete installation instructions |
| [Android Guide](docs/guides/ANDROID-AGENT-GUIDE.md) | Android-specific agent setup via Termux |
| [Design Spec](docs/design.md) | Original design specification |

## Project Structure

```
netsentinel/
├── core/                    # FastAPI server + analysis pipeline
│   ├── api/                 # REST + WebSocket endpoints
│   ├── pipeline/            # 5-layer analysis engine
│   ├── services/            # Agent manager, alerts, rules
│   └── ml/                  # Traffic classifier, anomaly detector
├── agent/                   # Device agent (laptop + Android)
│   ├── capture/             # Scapy, tcpdump, VPN-TUN modules
│   ├── processor.py         # Local flow assembly
│   └── transport.py         # WebSocket client
├── dashboard/               # Next.js web NOC dashboard
│   ├── src/app/             # Pages (overview, devices, traffic, threats)
│   └── src/components/      # React components
├── cli/                     # CLI tool (netsen)
├── bot/                     # Telegram alert bot
├── rules/                   # YAML threat detection rules
├── docs/                    # Documentation (you are here)
└── docker-compose.yml       # One-command deployment
```

## What Makes This a Technology

This isn't a tutorial project or a wrapper around existing tools. NetSentinel demonstrates:

1. **Protocol Design** — Custom binary protocol (MessagePack over WebSocket) between distributed agents and central server
2. **Pipeline Architecture** — 5-stage analysis pipeline processing packets through increasingly abstract layers
3. **Distributed Systems** — Agents on heterogeneous devices (Android, Linux, macOS) coordinated by a central brain
4. **Real-Time Streaming** — WebSocket-based live data from capture to dashboard in <2 seconds
5. **Security Engineering** — IDS/IPS capabilities with rule engine, threat intelligence, and automated response
6. **Applied ML** — Traffic classification and anomaly detection trained on real network data
7. **Infrastructure Control** — Agent-level traffic enforcement via iptables/nftables

## License

MIT

## Author

Ayush — Cybersecurity Enthusiast & Full-Stack Developer
