# NetSentinel — Technical Decisions Log

**Date:** 2026-04-09

This document records key technical decisions, the alternatives considered, and the rationale for each choice.

---

## TD-001: Core Server Framework

**Decision:** FastAPI (Python)

| Option | Pros | Cons |
|--------|------|------|
| **FastAPI** | Async-first, native WebSocket, Pydantic validation, fast development, Ayush's primary stack | Python GIL limits CPU parallelism |
| Go (net/http) | Superior performance, goroutines for concurrency | Slower development, weaker ecosystem for ML integration |
| Node.js (Express) | Good WebSocket support, fast I/O | Weak packet processing libraries, no Scapy equivalent |

**Rationale:** FastAPI's async capabilities handle the I/O-bound nature of packet ingestion perfectly. Python ecosystem (Scapy, scikit-learn, pyshark) is unmatched for network analysis + ML. The GIL is not a bottleneck because the workload is I/O-bound (WebSocket, DB writes), not CPU-bound.

---

## TD-002: Agent-Core Serialization

**Decision:** MessagePack over WebSocket

| Option | Pros | Cons |
|--------|------|------|
| **MessagePack + WebSocket** | Binary (30% smaller than JSON), fast encode/decode, persistent connection | Less human-readable for debugging |
| JSON + WebSocket | Human-readable, universal | Larger payloads, slower parse |
| gRPC | Strongly typed, streaming, efficient | Complex setup, overkill for 2-5 agents |
| MQTT | Designed for IoT, lightweight | Extra broker dependency, less control |

**Rationale:** MessagePack gives us binary efficiency without the complexity of gRPC. WebSocket provides persistent bidirectional communication needed for both data streaming (agent -> core) and command dispatch (core -> agent). For 2-5 agents, gRPC's complexity isn't justified.

---

## TD-003: Time-Series Database

**Decision:** TimescaleDB (PostgreSQL extension)

| Option | Pros | Cons |
|--------|------|------|
| **TimescaleDB** | PostgreSQL-compatible (SQL, joins, indexes), hypertables, automatic partitioning, built-in retention/compression, continuous aggregates | Heavier than pure time-series DBs |
| InfluxDB | Purpose-built for time-series, fast writes | Custom query language (Flux), no relational joins |
| ClickHouse | Extremely fast analytics | Complex setup, not great for small scale |
| Plain PostgreSQL | Simple, no extensions | Manual partitioning, no retention policies, slower at scale |

**Rationale:** TimescaleDB gives us the best of both worlds — full PostgreSQL compatibility (so we can JOIN packets with devices, flows with threats) plus time-series optimizations (hypertables, retention, compression, continuous aggregates). For a personal NOC, the analytical power of continuous aggregates alone justifies the choice.

---

## TD-004: Real-Time Cache

**Decision:** Redis

| Option | Pros | Cons |
|--------|------|------|
| **Redis** | Blazing fast, pub/sub for WebSocket fan-out, TTL for auto-expiry, sorted sets for threat ranking | Extra service to manage |
| In-memory (Python dict) | Zero latency, no dependency | Lost on restart, no pub/sub, no TTL |
| Memcached | Fast caching | No pub/sub, no data structures |

**Rationale:** Redis serves three roles: (1) real-time device state with TTL-based expiry, (2) pub/sub channels for WebSocket fan-out to multiple dashboard clients, (3) sorted sets for ranking active threats by severity. No alternative covers all three.

---

## TD-005: Frontend Framework

**Decision:** Next.js 15 + React 19 + TypeScript + Tailwind CSS

| Option | Pros | Cons |
|--------|------|------|
| **Next.js** | SSR for fast initial load, file-based routing, API routes if needed, React ecosystem | Heavier than SPA for simple dashboards |
| Vite + React SPA | Lighter, faster dev server | No SSR, manual routing |
| Vue + Nuxt | Good DX, similar capabilities | Different ecosystem from Ayush's stack |
| Grafana | Built for dashboards, no frontend code needed | Limited customization, no custom threat UI |

**Rationale:** Next.js provides SSR for the initial dashboard load (important when checking status quickly) and a mature React ecosystem for complex interactive components (packet inspector, flow tables, threat management). Tailwind CSS ensures consistent, responsive design without custom CSS.

---

## TD-006: Charting Library

**Decision:** Recharts (with MapLibre GL for geographic data)

| Option | Pros | Cons |
|--------|------|------|
| **Recharts** | React-native, composable, supports real-time updates, good docs | Not the fastest for 10K+ data points |
| D3.js | Most powerful, unlimited customization | Steep learning curve, not React-native |
| Chart.js | Lightweight, easy | Less React integration, limited customization |
| Apache ECharts | Very powerful, good for dashboards | Heavy bundle, non-React API |

**Rationale:** Recharts integrates naturally with React state and re-renders efficiently on WebSocket updates. For the data volumes of a personal NOC (hundreds of data points, not millions), Recharts is more than sufficient. MapLibre GL handles the geographic map separately — it's open-source and doesn't require a paid API key.

---

## TD-007: CLI Framework

**Decision:** Typer + Rich

| Option | Pros | Cons |
|--------|------|------|
| **Typer + Rich** | Type-safe CLI from type hints, Rich for beautiful TUI output (tables, progress bars, live updates), great DX | Python-only |
| Click | Battle-tested, widely used | More boilerplate than Typer, no TUI |
| argparse | Standard library, no deps | Verbose, ugly output |
| Go (cobra) | Fast binary, cross-platform | Separate codebase from Core |

**Rationale:** Typer generates CLI from type hints (minimal code), and Rich provides stunning terminal output — colored tables, live-updating panels, progress bars. Together they make a CLI that looks professional and is pleasant to use. Staying in Python keeps the CLI in the same codebase as Core.

---

## TD-008: Packet Capture Library

**Decision:** Scapy (primary) + tcpdump (Android fallback) + VPN-TUN (no-root)

| Option | Pros | Cons |
|--------|------|------|
| **Scapy** | Python-native, cross-platform, rich packet parsing, craft + capture | Slower than C-based alternatives |
| libpcap (via python-pcap) | Fast, industry standard | Less Pythonic, harder to parse |
| pyshark (tshark wrapper) | Wireshark-level parsing | Heavy dependency, subprocess overhead |
| tcpdump | Available on Android/Termux, fast | Raw output needs parsing |

**Rationale:** Scapy is the only Python library that can both capture and deeply parse packets without external dependencies. On Android (Termux), tcpdump is pre-packaged and more reliable. The VPN-TUN approach (creating a local VPN to intercept traffic) enables no-root Android capture — important for non-rooted phones.

---

## TD-009: ML Framework

**Decision:** scikit-learn + XGBoost

| Option | Pros | Cons |
|--------|------|------|
| **scikit-learn + XGBoost** | Lightweight, fast inference, perfect for tabular data (flow features), no GPU needed | Not suitable for deep learning |
| PyTorch | Flexible, state-of-the-art | Overkill for traffic classification, GPU preferred |
| TensorFlow | Production-ready | Heavy, overkill |
| No ML (rules only) | Simple | Misses novel patterns |

**Rationale:** Traffic classification and anomaly detection are tabular ML problems (features = numeric flow properties). Random Forest / XGBoost / Isolation Forest from scikit-learn are state-of-the-art for this. No GPU needed, inference takes <1ms per flow. Deep learning would add complexity without benefit.

---

## TD-010: Alert Delivery

**Decision:** Telegram Bot API

| Option | Pros | Cons |
|--------|------|------|
| **Telegram** | Free, reliable, inline buttons, rich formatting, available on all Ayush's devices | Requires internet, Telegram account |
| Discord | Webhooks are simple, rich embeds | Less suitable for personal alerts |
| Email (SMTP) | Universal | Slow, no inline actions, spam filters |
| Pushover | Purpose-built for notifications | Paid after trial |
| Ntfy | Self-hosted, free | Less rich UI, no inline buttons |

**Rationale:** Telegram's inline keyboard buttons enable acting on threats directly from the notification (Block, Whitelist, Dismiss) without opening the dashboard. It's free, reliable, and already on Ayush's phone. The Bot API is well-documented and python-telegram-bot is a mature library.

---

## TD-011: Threat Intelligence Feeds

**Decision:** abuse.ch + AlienVault OTX (free tiers)

| Feeds | What They Provide |
|-------|-------------------|
| abuse.ch URLhaus | Malware distribution URLs |
| abuse.ch Feodo Tracker | C2 server IPs (banking trojans) |
| abuse.ch SSL Blacklist | Malicious SSL certificate fingerprints |
| AlienVault OTX | Community-contributed IoCs (IPs, domains, hashes) |

**Rationale:** These are the most reliable free threat intelligence feeds. They cover the most critical categories (C2, malware, phishing) without requiring paid subscriptions. Updated daily, community-maintained, well-documented APIs.

---

## TD-012: Containerization Strategy

**Decision:** Docker Compose for services, bare Python for agents

**Services in Docker:** Core, Dashboard, TimescaleDB, Redis, Telegram Bot
**Not in Docker:** Device agents (run directly on laptops/phones)

**Rationale:** Agents need direct access to network interfaces for packet capture — running them in Docker would require `--net=host` and `--privileged`, defeating the purpose of containerization. Docker Compose for the server-side services gives one-command setup while agents install as regular Python packages on each device.
