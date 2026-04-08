# NetSentinel — Product Requirements Document

**Version:** 1.0
**Date:** 2026-04-09
**Author:** Ayush
**Status:** Approved

---

## 1. Executive Summary

NetSentinel is a personal Network Operations Center (NOC) — a distributed, multi-layered network monitoring and defense technology that gives complete visibility into network traffic across all personal devices (Android phones and laptops) with intelligent, tiered threat response.

Unlike simple network monitors, NetSentinel operates as a **full technology stack**: custom agent-server protocol, 5-layer deep analysis pipeline, real-time streaming architecture, ML-powered classification, and infrastructure-level traffic control. It is designed to be portfolio-grade — demonstrating mastery of networking, distributed systems, security, full-stack development, and machine learning.

---

## 2. Problem Statement

### The Gap
Personal device users have zero visibility into what their devices are doing on the network. Built-in tools (Android data usage, Activity Monitor) show surface-level stats — total MB used by an app. They don't answer:

- What IPs is my phone talking to right now?
- Is any app leaking data over unencrypted channels?
- Is someone ARP-spoofing my WiFi at this cafe?
- Which of my 3 devices is hogging bandwidth?
- Is there a suspicious persistent connection I don't recognize?

### The Opportunity
Enterprise NOCs solve this for corporations. NetSentinel brings that power to an individual — monitoring 2-5 personal devices with the same depth that a SOC analyst gets for a corporate network.

### Why This Matters for Portfolio
- Demonstrates **systems-level thinking** (not just app development)
- Shows **cybersecurity depth** (packet analysis, threat detection, IDS/IPS)
- Proves **distributed systems skills** (agents, real-time streaming, protocol design)
- Includes **ML integration** (traffic classification, anomaly detection)
- Full-stack implementation (backend, frontend, CLI, bot, database)

---

## 3. Target Users

### Primary: The Builder (Ayush)
- Cybersecurity enthusiast with multiple Android devices + laptop
- Wants deep network visibility across all personal devices
- Comfortable with terminal, Python, and networking concepts
- Uses devices across multiple networks (home WiFi, mobile data, public WiFi)

### Secondary: Security-Conscious Developers
- Developers who want to audit their own device traffic
- Students learning cybersecurity who need a hands-on platform
- Privacy-focused users who want to know exactly what their devices transmit

---

## 4. User Stories

### Device Monitoring
| ID | Story | Priority |
|----|-------|----------|
| U1 | As a user, I want to see all my devices and their online/offline status in one place | P0 |
| U2 | As a user, I want to see real-time bandwidth usage per device | P0 |
| U3 | As a user, I want to see which apps are using network on each device | P0 |
| U4 | As a user, I want to see every active TCP/UDP connection on a device | P0 |
| U5 | As a user, I want to drill down from app -> connections -> packets | P1 |

### Traffic Analysis
| ID | Story | Priority |
|----|-------|----------|
| T1 | As a user, I want to see which IPs/domains my devices talk to most | P0 |
| T2 | As a user, I want to see traffic broken down by protocol (HTTP, DNS, TLS, etc.) | P1 |
| T3 | As a user, I want geographic visualization of where my traffic goes | P1 |
| T4 | As a user, I want to see bandwidth over time (hourly, daily, weekly) | P0 |
| T5 | As a user, I want to export packet captures to .pcap for Wireshark analysis | P2 |

### Threat Detection
| ID | Story | Priority |
|----|-------|----------|
| S1 | As a user, I want to be alerted when my device connects to a known malicious IP | P0 |
| S2 | As a user, I want to detect ARP spoofing on my WiFi network | P0 |
| S3 | As a user, I want to detect DNS tunneling attempts | P1 |
| S4 | As a user, I want to know if any app sends credentials over plaintext HTTP | P0 |
| S5 | As a user, I want to detect rogue access points (evil twin attacks) | P1 |
| S6 | As a user, I want anomaly detection that learns my normal traffic and flags deviations | P1 |

### Response Actions
| ID | Story | Priority |
|----|-------|----------|
| R1 | As a user, I want low-severity events to be silently logged | P0 |
| R2 | As a user, I want medium-severity events to alert me via Telegram with recommendations | P0 |
| R3 | As a user, I want high-severity events to auto-block the connection and alert me | P0 |
| R4 | As a user, I want critical events to quarantine the device and save forensic data | P1 |
| R5 | As a user, I want to manually block/whitelist IPs and domains | P0 |

### Multi-Interface Access
| ID | Story | Priority |
|----|-------|----------|
| I1 | As a user, I want a web dashboard for visual monitoring | P0 |
| I2 | As a user, I want a CLI tool for quick terminal-based inspection | P0 |
| I3 | As a user, I want Telegram alerts on my phone when threats are detected | P0 |
| I4 | As a user, I want a daily summary of network activity and threats | P2 |

---

## 5. Functional Requirements

### FR-1: Device Agent System
- **FR-1.1:** Python-based agent runs on Linux/macOS laptops and Android (via Termux)
- **FR-1.2:** Agent captures packets using Scapy (laptop) or tcpdump/VPN-TUN (Android)
- **FR-1.3:** Agent performs local flow assembly to reduce data sent to Core
- **FR-1.4:** Agent connects to Core via WebSocket with MessagePack serialization
- **FR-1.5:** Agent authenticates with pre-shared API key over TLS
- **FR-1.6:** Agent executes block/allow commands received from Core
- **FR-1.7:** Agent sends heartbeats with device metrics (battery, CPU, network type)
- **FR-1.8:** Agent auto-reconnects on connection loss with exponential backoff

### FR-2: Core Analysis Pipeline
- **FR-2.1:** Layer 1 (Packet Engine) processes raw packets, extracts headers and metadata
- **FR-2.2:** Layer 2 (Flow Analyzer) assembles packets into TCP sessions and UDP flows
- **FR-2.3:** Layer 3 (App Classifier) identifies applications using port mapping, DNS correlation, DPI, and ML
- **FR-2.4:** Layer 4 (Threat Detector) evaluates flows against rule engine and anomaly models
- **FR-2.5:** Layer 5 (Response Engine) executes tiered actions based on threat severity
- **FR-2.6:** Pipeline processes packets to dashboard in under 2 seconds end-to-end

### FR-3: Data Storage
- **FR-3.1:** TimescaleDB stores packet metadata and flow records as time-series data
- **FR-3.2:** Redis maintains real-time device state, active flows, and live bandwidth counters
- **FR-3.3:** Threat rules stored in PostgreSQL with YAML import/export
- **FR-3.4:** Automatic data retention: raw packets 7 days, flows 90 days, threats indefinitely

### FR-4: Web Dashboard
- **FR-4.1:** Overview page with device grid, aggregate bandwidth, threat count
- **FR-4.2:** Device detail page with app breakdown, connection list, packet inspector
- **FR-4.3:** Traffic analysis page with flow graph, top talkers, geographic map
- **FR-4.4:** Threat center with severity filtering, one-click actions, forensic detail
- **FR-4.5:** Settings page for device management, rule editor, notification config
- **FR-4.6:** All visualizations update in real-time via WebSocket

### FR-5: CLI Tool
- **FR-5.1:** Device listing and inspection commands
- **FR-5.2:** Live traffic monitoring with TUI (Rich library)
- **FR-5.3:** Flow and packet querying with filters
- **FR-5.4:** Threat listing and manual actions (block, whitelist, quarantine)
- **FR-5.5:** Statistics and reporting commands
- **FR-5.6:** PCAP export capability

### FR-6: Alert System
- **FR-6.1:** Telegram bot sends tiered severity notifications
- **FR-6.2:** Inline action buttons on alerts (Block, Whitelist, Investigate, Dismiss)
- **FR-6.3:** Configurable notification preferences (which severities, quiet hours)
- **FR-6.4:** Daily summary report via Telegram
- **FR-6.5:** Bot commands for quick status checks (/status, /devices, /threats)

---

## 6. Non-Functional Requirements

### Performance
| Metric | Target |
|--------|--------|
| Packet-to-dashboard latency | < 2 seconds |
| Concurrent device agents | 5+ without degradation |
| Dashboard page load | < 1.5 seconds |
| WebSocket update frequency | 1 Hz for stats, real-time for threats |
| Packet capture loss | < 5% at normal traffic volumes |
| CLI response time | < 500ms for queries |
| Telegram alert delivery | < 5 seconds from detection |

### Reliability
- Agent auto-reconnects within 30 seconds of connection loss
- Core server handles agent disconnection gracefully (marks device offline)
- No data loss for high-severity threat events (persisted before alerting)
- Database handles 1M+ flow records without query degradation

### Security
- All agent-core communication over TLS
- Pre-shared API key authentication for agents
- JWT authentication for dashboard and API
- No data leaves local network (fully self-hosted)
- Packet data encrypted at rest (TimescaleDB encryption)
- API keys rotatable without agent restart

### Scalability
- Designed for 2-5 devices (personal use)
- Architecture supports horizontal scaling if needed (multiple Core instances behind load balancer)
- TimescaleDB auto-partitioning handles growing data volumes

---

## 7. Constraints

- **Android capture requires either root or VPN-TUN approach** — document both paths
- **Termux environment has limited resources** — agent must be lightweight (<50MB RAM)
- **Free threat intelligence feeds only** — no paid subscriptions
- **Self-hosted only** — no cloud dependency
- **Single-user system** — no multi-tenancy needed

---

## 8. Out of Scope

- iOS device support (no Termux equivalent)
- Cloud-hosted deployment
- Multi-user / team features
- Commercial threat intelligence integration
- VPN / traffic routing functionality
- Web-based packet replay
- Automated penetration testing

---

## 9. Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Device agent uptime | > 99% during active use | Heartbeat logs |
| Threat detection accuracy | > 90% on test scenarios | Run simulated attacks |
| False positive rate | < 10% | Review threat logs weekly |
| End-to-end latency | < 2s (P95) | Timestamp comparison |
| Portfolio impact | Demonstrable in interviews | Live demo capability |

---

## 10. Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Android packet capture requires root | Blocks Android agent | Medium | Implement VPN-TUN no-root alternative |
| Termux resource constraints | Agent crashes on phone | Medium | Aggressive local filtering, battery-aware mode |
| False positive alerts | Alert fatigue | High | Tunable thresholds, whitelist system, ML baseline |
| TimescaleDB storage growth | Disk fills up | Medium | Auto-retention policies, compression |
| Complex setup process | Hard to demo | Low | Docker Compose one-command setup |

---

## 11. Dependencies

| Dependency | Purpose | Risk |
|------------|---------|------|
| Termux (Android) | Run Python agent on Android | Maintained by open-source community |
| Scapy | Packet capture/crafting | Stable, well-maintained |
| TimescaleDB | Time-series storage | PostgreSQL extension, battle-tested |
| MaxMind GeoLite2 | IP geolocation | Free tier, requires account |
| abuse.ch / AlienVault OTX | Threat intelligence feeds | Free, community-maintained |
| Telegram Bot API | Alert delivery | Stable, free |

---

## 12. Timeline Estimate

| Phase | Scope | Duration |
|-------|-------|----------|
| Phase 1 — Foundation | Core server, basic agent, database, packet capture | 2-3 weeks |
| Phase 2 — Pipeline | 5-layer analysis pipeline, flow assembly, basic rules | 2-3 weeks |
| Phase 3 — Dashboard | Web NOC dashboard with real-time visualization | 2 weeks |
| Phase 4 — CLI + Alerts | CLI tool, Telegram bot, tiered response | 1-2 weeks |
| Phase 5 — ML + Polish | Traffic classifier, anomaly detection, threat feeds | 2 weeks |
| Phase 6 — Hardening | Testing, documentation, Docker setup, demo prep | 1 week |
| **Total** | **Full technology** | **10-13 weeks** |

---

## Appendix A: Glossary

| Term | Definition |
|------|-----------|
| NOC | Network Operations Center — centralized monitoring hub |
| IDS | Intrusion Detection System — detects malicious activity |
| IPS | Intrusion Prevention System — detects and blocks malicious activity |
| DPI | Deep Packet Inspection — analyzing packet payloads beyond headers |
| Flow | A logical network conversation (TCP session or UDP stream) |
| ARP Spoofing | Attack where attacker links their MAC to victim's IP |
| DNS Tunneling | Encoding data in DNS queries to exfiltrate information |
| C2 | Command and Control — server used by malware to receive instructions |
| MITM | Man-in-the-Middle — attacker intercepts communication |
| TUN | Virtual network tunnel interface |
