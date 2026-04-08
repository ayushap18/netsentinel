# NetSentinel — Security Architecture

**Date:** 2026-04-09

---

## 1. Threat Model

NetSentinel is a security tool — it must itself be secure. This document covers how we protect the system from misuse and compromise.

### Assets to Protect
| Asset | Sensitivity | Impact if Compromised |
|-------|------------|----------------------|
| Packet data | HIGH | Complete network traffic visibility |
| Device agent access | HIGH | Remote code execution on devices |
| Threat rules | MEDIUM | Attacker could disable detection |
| API credentials | HIGH | Full system control |
| Forensic dumps | HIGH | Evidence of network activity |

### Attack Surface
| Surface | Threat | Mitigation |
|---------|--------|------------|
| Agent-Core WebSocket | Man-in-the-middle, replay | TLS + API key auth |
| REST API | Unauthorized access | JWT + rate limiting |
| Dashboard | Session hijacking, XSS | HttpOnly cookies, CSP headers |
| Telegram Bot | Command injection | Whitelist chat ID, input validation |
| Database | SQL injection, data theft | Parameterized queries, encrypted volume |
| Agent on device | Privilege escalation | Minimal permissions, no root daemon |

---

## 2. Authentication & Authorization

### Agent Authentication
```
Agent ──── TLS 1.3 ────▶ Core
         │
         ├── API key in WebSocket handshake
         │   Header: X-Agent-Key: ns_ak_<random_64_hex>
         │
         └── Key validated against bcrypt hash in DB
```

- Keys generated during `netsen-agent setup`
- Stored hashed (bcrypt) in Core database
- Keys are rotatable: `POST /api/auth/rotate-agent-key/{device_id}`
- Rejected agents get `4401 Unauthorized` and connection is closed
- Failed auth attempts logged with source IP

### Dashboard/API Authentication
```
User ──── Login ────▶ JWT Access Token (1h TTL)
                       + Refresh Token (7d TTL, HttpOnly cookie)
```

- Password stored as bcrypt hash
- Access token: short-lived (1 hour), in Authorization header
- Refresh token: longer-lived (7 days), HttpOnly secure cookie
- Token rotation on refresh (old refresh token invalidated)
- Rate limit: 5 failed login attempts → 15 minute lockout

### Telegram Bot Authentication
- Bot only responds to pre-configured chat IDs (whitelist)
- Unknown chat IDs receive: "Unauthorized. This bot is private."
- All bot commands logged with chat ID

---

## 3. Encryption

### In Transit
| Channel | Protocol | Min Version |
|---------|----------|-------------|
| Agent → Core | WSS (WebSocket over TLS) | TLS 1.3 |
| Browser → Dashboard | HTTPS | TLS 1.2 |
| CLI → API | HTTPS | TLS 1.2 |
| Core → Telegram | HTTPS (Bot API) | TLS 1.2 |

### At Rest
| Data | Encryption | Method |
|------|-----------|--------|
| TimescaleDB | Volume encryption | Docker volume with LUKS or macOS FileVault |
| Forensic .pcap dumps | File-level encryption | AES-256-GCM, key derived from master password |
| Agent config (API key) | File permissions | 600 (owner read/write only) |
| .env secrets | File permissions | 600, never committed to git |

---

## 4. Network Security

### Firewall Rules (Core Server)
```
# Only allow:
- Agent WebSocket connections (port 8000, from known device IPs or LAN)
- Dashboard access (port 3000, localhost or LAN)
- Telegram API outbound (HTTPS to api.telegram.org)

# Block:
- All inbound from public internet (unless explicitly exposed)
- Direct database access from non-Core sources
```

### Agent Network Permissions
```
# Agent needs:
- CAP_NET_RAW (packet capture) — or sudo/root
- Outbound WebSocket to Core server
- Local iptables/nftables access (for enforcement)

# Agent must NOT:
- Listen on any port (no inbound connections)
- Access other devices' traffic (only its own interfaces)
- Store unencrypted packet dumps
```

---

## 5. Input Validation

### API Input
- All request bodies validated via Pydantic models (strict types)
- SQL queries use SQLAlchemy ORM (parameterized, never raw SQL with user input)
- WebSocket messages validated against MessagePack schema
- File uploads (rule YAML) parsed with safe YAML loader (`yaml.safe_load`)

### Rule Engine
- YAML rules parsed with `yaml.safe_load` (no arbitrary code execution)
- Rule conditions are declarative (JSON-like), not executable code
- Regex patterns in rules are pre-compiled with timeout (ReDoS protection)
- Max rule condition depth: 5 levels (prevents recursive bombs)

### Telegram Bot
- Command arguments sanitized (no shell injection)
- IP addresses validated with `ipaddress` module
- Domain names validated against RFC 1035
- All user input escaped before display

---

## 6. Secrets Management

### Secrets Inventory
| Secret | Storage | Rotation |
|--------|---------|----------|
| Database password | `.env` file (600 perms) | On compromise |
| Redis password | `.env` file | On compromise |
| Agent API keys | bcrypt hash in DB, plaintext in agent config | Anytime via API |
| JWT signing key | `.env` file | Monthly recommended |
| Telegram bot token | `.env` file | On compromise |
| Dashboard admin password | bcrypt hash in DB | User-initiated |

### .env Template
```bash
# Core
DATABASE_URL=postgresql://netsentinel:CHANGE_ME@timescaledb:5432/netsentinel
REDIS_URL=redis://:CHANGE_ME@redis:6379/0
JWT_SECRET=GENERATE_RANDOM_64_CHAR
MASTER_ENCRYPTION_KEY=GENERATE_RANDOM_32_BYTE_HEX

# Admin
ADMIN_USERNAME=admin
ADMIN_PASSWORD_HASH=<bcrypt_hash>

# Telegram
TELEGRAM_BOT_TOKEN=<from_botfather>
TELEGRAM_ALLOWED_CHAT_IDS=123456789,987654321

# Agent (generated during setup)
# Stored in ~/.netsentinel/agent.conf on each device
```

### Git Protection
```gitignore
# Never commit:
.env
.env.*
*.pem
*.key
agent.conf
forensics/
ml_models/*.pkl
```

---

## 7. Logging & Audit Trail

### Security Events Logged
| Event | Log Level | Details Captured |
|-------|-----------|-----------------|
| Agent connected | INFO | device_id, source_ip, agent_version |
| Agent auth failed | WARN | source_ip, provided_key_prefix |
| Dashboard login | INFO | username, source_ip |
| Dashboard login failed | WARN | username, source_ip, attempt_count |
| Threat detected | INFO | threat_id, severity, device_id, rule_id |
| Block action | INFO | target_ip/domain, initiated_by, device_id |
| Quarantine action | WARN | device_id, initiated_by, reason |
| Rule modified | INFO | rule_id, change_type, modified_by |
| Agent key rotated | INFO | device_id, initiated_by |

### Log Storage
- Application logs: stdout (Docker captures to JSON files)
- Security audit log: separate file (`/var/log/netsentinel/audit.log`)
- Log retention: 30 days (configurable)
- No sensitive data in logs (no API keys, passwords, packet payloads)

---

## 8. Data Privacy

### What NetSentinel Captures
- Packet headers (IP, port, protocol, flags)
- DNS queries (domain names)
- TLS SNI (server names)
- Flow metadata (bytes, duration, timestamps)
- Limited DPI (pattern matching, not full payload storage)

### What NetSentinel Does NOT Capture
- Full packet payloads (unless flagged as threat → forensic dump)
- Decrypted HTTPS content
- Passwords or authentication tokens (detected and alerted, not stored)
- Personal data beyond network metadata

### Data Residency
- All data stays on the local machine / local network
- No telemetry, no cloud sync, no external reporting
- Forensic dumps are encrypted and auto-deleted after 30 days
- User can purge all data: `netsen admin purge --confirm`

---

## 9. Hardening Checklist

### Before First Run
- [ ] Change all default passwords in `.env`
- [ ] Generate random JWT secret (64+ characters)
- [ ] Generate random master encryption key
- [ ] Set up Telegram bot and whitelist chat IDs
- [ ] Enable disk encryption on host machine
- [ ] Restrict firewall to LAN only

### Ongoing
- [ ] Update threat feeds weekly (automated via cron)
- [ ] Review threat logs for false positives
- [ ] Rotate agent API keys quarterly
- [ ] Update Docker images monthly
- [ ] Review audit logs for anomalies
- [ ] Test backup/restore procedure
