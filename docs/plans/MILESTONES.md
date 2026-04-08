# NetSentinel — Milestones & Demo Checkpoints

**Date:** 2026-04-09

Each milestone is a **demo-able checkpoint** — a point where you can show something working end-to-end. This is important for portfolio demonstrations and for maintaining momentum.

---

## Milestone Map

```
M1 ──▶ M2 ──▶ M3 ──▶ M4 ──▶ M5 ──▶ M6 ──▶ M7 ──▶ M8
│       │       │       │       │       │       │       │
Heartbeat  Packets  Flows   Threats  Dashboard  CLI    ML    Demo
Online    Captured  Visible Detected  Live      Works  Smart Ready
```

---

## M1: First Heartbeat (End of Week 2)

**Demo:** Start agent on laptop → Core shows device "online" → stop agent → device goes "offline".

**What's working:**
- Docker Compose (Core + TimescaleDB + Redis)
- Agent boots, connects via WebSocket
- Heartbeat system (30s interval)
- Device registration + status tracking
- `GET /api/devices` returns online device

**You can say:** "My distributed agent system is communicating with the central server in real-time with a custom binary protocol."

---

## M2: First Packet Captured (End of Week 3)

**Demo:** Open browser on laptop → packets appear in database → query via API.

**What's working:**
- Scapy captures packets on laptop
- Packets serialized with MessagePack, sent to Core
- L1 Packet Engine processes and stores in TimescaleDB
- `GET /api/devices/{id}/packets` returns captured packets
- Packet filtering (protocol, IP, port)

**You can say:** "The system captures raw network packets, processes them through a packet engine, and stores them as time-series data."

---

## M3: Flows + Apps Visible (End of Week 4)

**Demo:** Use Instagram on phone → system shows "instagram" flow with bytes, duration, destination country.

**What's working:**
- L2 Flow Analyzer assembles packets into flows
- L3 App Classifier identifies apps (port + DNS + TLS SNI)
- GeoIP maps destinations to countries
- `GET /api/devices/{id}/flows` with app/country data
- `GET /api/stats/apps` shows per-app bandwidth

**You can say:** "The pipeline assembles raw packets into logical network flows and classifies them by application using deep packet inspection and DNS correlation."

---

## M4: First Threat Detected (End of Week 6)

**Demo:** Simulate DNS tunnel / connect to test C2 IP → threat appears → agent auto-blocks the IP.

**What's working:**
- L4 Threat Detector with rule engine
- L5 Tiered Response (log, alert, block, quarantine)
- Threat feeds loaded (abuse.ch)
- Agent enforcer (iptables block)
- Forensic packet dump saved
- `GET /api/threats` returns detected threats

**You can say:** "The system detected a DNS tunneling attempt, classified it as high severity, automatically blocked the connection on the device, and saved forensic evidence."

---

## M5: Dashboard Live (End of Week 8)

**Demo:** Open browser → see all devices, click into one → see live traffic, app breakdown, click a flow → see packets. Switch to Threat Center → see detected threats with action buttons.

**What's working:**
- Full web NOC dashboard
- Real-time WebSocket updates
- Device grid with live status
- Per-device drill-down (apps → flows → packets)
- Traffic analysis page with charts and geo map
- Threat center with severity filtering and actions
- Settings page

**You can say:** "This is the NOC dashboard — every graph updates in real-time via WebSocket. I can drill down from a device to an app to an individual packet."

---

## M6: CLI Operational (End of Week 9)

**Demo:** Run `netsen monitor` → live TUI shows traffic. Run `netsen threats` → table of threats. Run `netsen block <ip>` → blocked across all devices.

**What's working:**
- Full CLI tool with Rich TUI
- Live monitoring mode
- Flow and packet querying
- Threat management
- Block/whitelist commands

**You can say:** "For power users, there's a CLI that gives the same depth as the dashboard. Here's the live monitoring TUI."

---

## M7: Smart Detection (End of Week 12)

**Demo:** Normal traffic → no alerts. Run anomalous pattern → ML flags it as unusual. Show classification confidence scores.

**What's working:**
- ML traffic classifier (Random Forest)
- Anomaly detector (Isolation Forest)
- Classification confidence in dashboard
- Anomaly scores on flows
- Auto-updating threat feeds

**You can say:** "The system learned my normal traffic patterns over two weeks and now flags genuine anomalies. The ML classifier identifies applications with 85%+ accuracy."

---

## M8: Demo Ready (End of Week 13)

**Demo:** Full end-to-end demo script with simulated traffic and attacks.

**Demo script:**
1. Start system: `docker-compose up -d`
2. Connect 2 agents (laptop + phone)
3. Show dashboard: devices online, live traffic
4. Browse web on phone → show flows appearing in real-time
5. Open Instagram → show app classified, bandwidth tracked
6. Show geographic map of connections
7. Trigger simulated attack → threat detected → auto-blocked → Telegram alert
8. Show forensic evidence in dashboard
9. Use CLI: `netsen threats`, `netsen monitor`
10. Show daily Telegram summary

**What's working:** Everything.

**You can say:** "NetSentinel is a personal Network Operations Center. It's a distributed system with device agents, a 5-layer analysis pipeline, ML-powered classification, and automated threat response. Every component — from the custom binary protocol to the real-time dashboard — was built from scratch."

---

## Portfolio Talking Points Per Milestone

| Milestone | Skill Demonstrated |
|-----------|-------------------|
| M1 | Distributed systems, WebSocket protocol design, Docker |
| M2 | Low-level networking (raw packet capture), time-series databases |
| M3 | Deep packet inspection, DNS analysis, GeoIP, data pipeline |
| M4 | Security engineering (IDS/IPS), threat intelligence, automated response |
| M5 | Full-stack development, real-time web apps, data visualization |
| M6 | CLI design, TUI development, developer tooling |
| M7 | Machine learning (classification + anomaly detection), applied ML |
| M8 | Systems integration, documentation, demo skills |
