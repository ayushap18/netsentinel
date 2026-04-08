# NetSentinel — Data Model & Database Design

**Version:** 1.0
**Date:** 2026-04-09

---

## 1. Database Strategy

| Database | Purpose | Data Type |
|----------|---------|-----------|
| **TimescaleDB** | Persistent storage for time-series and relational data | Packets, flows, devices, threats, rules |
| **Redis** | Real-time state and pub/sub | Live device status, active flows, bandwidth counters, WebSocket fan-out |

---

## 2. Entity Relationship Diagram

```
┌──────────────┐       ┌──────────────────┐       ┌───────────────┐
│   devices    │       │     flows        │       │   packets     │
│──────────────│       │──────────────────│       │───────────────│
│ id (PK)      │◄──┐   │ id (PK, UUID)    │◄──┐   │ id (PK)       │
│ name         │   │   │ device_id (FK)───┘   │   │ device_id (FK)│
│ device_type  │   │   │ start_time       │   │   │ timestamp     │
│ last_seen    │   │   │ end_time         │   │   │ src_ip        │
│ network_type │   │   │ src_ip           │   │   │ dst_ip        │
│ local_ip     │   │   │ dst_ip           │   └───│ flow_id (FK)  │
│ public_ip    │   │   │ src_port         │       │ src_port      │
│ status       │   │   │ dst_port         │       │ dst_port      │
│ metadata     │   │   │ protocol         │       │ protocol      │
└──────────────┘   │   │ total_bytes_in   │       │ payload_size  │
                   │   │ total_bytes_out  │       │ flags         │
                   │   │ packet_count     │       │ raw_summary   │
                   │   │ retransmissions  │       └───────────────┘
                   │   │ avg_rtt_ms       │
                   │   │ app_name         │
                   │   │ app_category     │       ┌───────────────┐
                   │   │ dns_name         │       │ threat_rules  │
                   │   │ country_code     │       │───────────────│
                   │   │ status           │       │ id (PK)       │
                   │   └──────────────────┘       │ name          │
                   │                               │ description   │
                   │   ┌──────────────────┐       │ category      │
                   │   │    threats       │       │ severity      │
                   │   │──────────────────│       │ condition     │
                   └───│ device_id (FK)   │   ┌──│ action        │
                       │ id (PK)         │   │  │ enabled       │
                       │ flow_id (FK)    │   │  │ created_at    │
                       │ detected_at     │   │  └───────────────┘
                       │ severity        │   │
                       │ category        │   │
                       │ description     │   │
                       │ evidence        │   │
                       │ rule_id (FK)────┘   │
                       │ status          │
                       │ action_taken    │
                       │ resolved_at     │
                       └──────────────────┘
```

---

## 3. Table Definitions

### 3.1 `devices` — Registered Devices

```sql
CREATE TABLE devices (
    id            TEXT PRIMARY KEY,              -- e.g., "phone-1", "laptop"
    name          TEXT NOT NULL,                 -- Human-friendly name
    device_type   TEXT NOT NULL,                 -- "android", "laptop", "pc"
    last_seen     TIMESTAMPTZ,                  -- Last heartbeat timestamp
    network_type  TEXT,                          -- "wifi", "4g", "5g", "ethernet"
    local_ip      INET,                         -- Local network IP
    public_ip     INET,                         -- Public-facing IP
    status        TEXT DEFAULT 'offline',        -- "online", "offline", "quarantined"
    agent_version TEXT,                          -- Agent software version
    os_info       TEXT,                          -- "Android 14", "Ubuntu 24.04", etc.
    metadata      JSONB DEFAULT '{}',           -- Battery level, CPU usage, etc.
    registered_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_devices_status ON devices(status);
```

### 3.2 `packets` — Raw Packet Metadata (Hypertable)

```sql
CREATE TABLE packets (
    id            BIGSERIAL,
    device_id     TEXT NOT NULL REFERENCES devices(id),
    timestamp     TIMESTAMPTZ NOT NULL,
    src_ip        INET NOT NULL,
    dst_ip        INET NOT NULL,
    src_port      INT,
    dst_port      INT,
    protocol      TEXT NOT NULL,                -- "TCP", "UDP", "ICMP", "DNS", etc.
    payload_size  INT NOT NULL DEFAULT 0,
    flags         TEXT,                          -- TCP flags: "SYN", "SYN-ACK", "FIN", etc.
    direction     TEXT,                          -- "inbound", "outbound"
    flow_id       UUID,                          -- Links to parent flow
    raw_summary   JSONB,                        -- Additional metadata (TLS version, SNI, etc.)
    PRIMARY KEY (id, timestamp)
);

SELECT create_hypertable('packets', 'timestamp',
    chunk_time_interval => INTERVAL '1 hour'
);

-- Retention: auto-delete after 7 days
SELECT add_retention_policy('packets', INTERVAL '7 days');

-- Compression after 1 day
SELECT add_compression_policy('packets', INTERVAL '1 day');

-- Indexes for common queries
CREATE INDEX idx_packets_device ON packets(device_id, timestamp DESC);
CREATE INDEX idx_packets_flow ON packets(flow_id);
CREATE INDEX idx_packets_dst_ip ON packets(dst_ip, timestamp DESC);
CREATE INDEX idx_packets_protocol ON packets(protocol, timestamp DESC);
```

### 3.3 `flows` — Network Flows (Hypertable)

```sql
CREATE TABLE flows (
    id              UUID DEFAULT gen_random_uuid(),
    device_id       TEXT NOT NULL REFERENCES devices(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    src_ip          INET NOT NULL,
    dst_ip          INET NOT NULL,
    src_port        INT,
    dst_port        INT,
    protocol        TEXT NOT NULL,
    total_bytes_in  BIGINT DEFAULT 0,
    total_bytes_out BIGINT DEFAULT 0,
    packet_count    INT DEFAULT 0,
    retransmissions INT DEFAULT 0,
    avg_rtt_ms      FLOAT,
    app_name        TEXT,                        -- Classified app name
    app_category    TEXT,                        -- "social", "streaming", "system", etc.
    dns_name        TEXT,                        -- Resolved domain (e.g., "instagram.com")
    tls_version     TEXT,                        -- "TLS 1.2", "TLS 1.3", etc.
    tls_sni         TEXT,                        -- Server Name Indication
    country_code    CHAR(2),                     -- GeoIP country code
    city            TEXT,                        -- GeoIP city
    asn             INT,                         -- Autonomous System Number
    as_org          TEXT,                        -- AS organization name
    status          TEXT DEFAULT 'active',       -- "active", "completed", "blocked"
    threat_id       INT,                         -- If flagged as threat
    PRIMARY KEY (id, start_time)
);

SELECT create_hypertable('flows', 'start_time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Retention: auto-delete after 90 days
SELECT add_retention_policy('flows', INTERVAL '90 days');

-- Compression after 7 days
SELECT add_compression_policy('flows', INTERVAL '7 days');

-- Indexes
CREATE INDEX idx_flows_device ON flows(device_id, start_time DESC);
CREATE INDEX idx_flows_app ON flows(app_name, start_time DESC);
CREATE INDEX idx_flows_dst ON flows(dst_ip, start_time DESC);
CREATE INDEX idx_flows_dns ON flows(dns_name, start_time DESC);
CREATE INDEX idx_flows_status ON flows(status) WHERE status = 'active';
CREATE INDEX idx_flows_category ON flows(app_category, start_time DESC);
```

### 3.4 `threats` — Detected Threats

```sql
CREATE TABLE threats (
    id            SERIAL PRIMARY KEY,
    device_id     TEXT NOT NULL REFERENCES devices(id),
    flow_id       UUID,
    detected_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    severity      TEXT NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    category      TEXT NOT NULL,                 -- "malware_c2", "dns_tunnel", "arp_spoof", etc.
    title         TEXT NOT NULL,                 -- Short description for display
    description   TEXT,                          -- Detailed explanation
    evidence      JSONB NOT NULL DEFAULT '{}',  -- Captured data supporting detection
    rule_id       INT REFERENCES threat_rules(id),
    status        TEXT DEFAULT 'active'
                  CHECK (status IN ('active', 'blocked', 'dismissed', 'whitelisted', 'resolved')),
    action_taken  TEXT,                          -- "logged", "alerted", "blocked", "quarantined"
    resolved_at   TIMESTAMPTZ,
    resolved_by   TEXT,                          -- "auto", "manual", "whitelist"
    forensic_path TEXT,                          -- Path to saved pcap dump
    metadata      JSONB DEFAULT '{}'
);

CREATE INDEX idx_threats_device ON threats(device_id, detected_at DESC);
CREATE INDEX idx_threats_severity ON threats(severity, detected_at DESC);
CREATE INDEX idx_threats_status ON threats(status) WHERE status = 'active';
CREATE INDEX idx_threats_category ON threats(category);
```

### 3.5 `threat_rules` — Detection Rules

```sql
CREATE TABLE threat_rules (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    description TEXT,
    category    TEXT NOT NULL,                   -- "network", "dns", "application", "anomaly"
    severity    TEXT NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    rule_type   TEXT NOT NULL,                   -- "ip_match", "dns_entropy", "dpi_pattern", etc.
    condition   JSONB NOT NULL,                  -- Rule conditions
    action      TEXT NOT NULL,                   -- "log", "alert", "block", "quarantine"
    notify      BOOLEAN DEFAULT FALSE,
    enabled     BOOLEAN DEFAULT TRUE,
    hit_count   BIGINT DEFAULT 0,               -- How many times rule has triggered
    last_hit    TIMESTAMPTZ,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_rules_enabled ON threat_rules(enabled) WHERE enabled = TRUE;
CREATE INDEX idx_rules_category ON threat_rules(category);
```

### 3.6 `bandwidth_stats` — Aggregated Bandwidth (Hypertable)

```sql
CREATE TABLE bandwidth_stats (
    device_id     TEXT NOT NULL REFERENCES devices(id),
    timestamp     TIMESTAMPTZ NOT NULL,
    interval_sec  INT NOT NULL DEFAULT 60,       -- Aggregation interval
    bytes_in      BIGINT DEFAULT 0,
    bytes_out     BIGINT DEFAULT 0,
    packet_count  INT DEFAULT 0,
    flow_count    INT DEFAULT 0,
    active_apps   JSONB,                         -- {"instagram": 1234, "chrome": 5678}
    network_type  TEXT,                          -- Network at time of measurement
    PRIMARY KEY (device_id, timestamp)
);

SELECT create_hypertable('bandwidth_stats', 'timestamp',
    chunk_time_interval => INTERVAL '1 day'
);

SELECT add_retention_policy('bandwidth_stats', INTERVAL '30 days');
```

### 3.7 `blocklist` — Blocked IPs/Domains

```sql
CREATE TABLE blocklist (
    id          SERIAL PRIMARY KEY,
    entry_type  TEXT NOT NULL CHECK (entry_type IN ('ip', 'domain', 'cidr')),
    value       TEXT NOT NULL,                   -- IP, domain, or CIDR range
    reason      TEXT,
    source      TEXT,                            -- "manual", "auto_rule", "threat_feed"
    threat_id   INT REFERENCES threats(id),
    added_at    TIMESTAMPTZ DEFAULT NOW(),
    expires_at  TIMESTAMPTZ,                     -- NULL = permanent
    active      BOOLEAN DEFAULT TRUE,
    UNIQUE(entry_type, value)
);

CREATE INDEX idx_blocklist_active ON blocklist(entry_type, active) WHERE active = TRUE;
```

### 3.8 `whitelist` — Whitelisted IPs/Domains

```sql
CREATE TABLE whitelist (
    id          SERIAL PRIMARY KEY,
    entry_type  TEXT NOT NULL CHECK (entry_type IN ('ip', 'domain', 'cidr')),
    value       TEXT NOT NULL,
    reason      TEXT,
    added_at    TIMESTAMPTZ DEFAULT NOW(),
    active      BOOLEAN DEFAULT TRUE,
    UNIQUE(entry_type, value)
);
```

---

## 4. Redis Schema

### 4.1 Device State

```
# Device online status + metrics
device:{device_id}:status
    HASH {
        status: "online" | "offline" | "quarantined",
        last_heartbeat: "2026-04-09T10:30:00Z",
        network_type: "wifi",
        local_ip: "192.168.1.105",
        battery: "78",
        cpu_percent: "12.5",
        agent_uptime: "3600"
    }
    TTL: 120s (auto-expires if no heartbeat)
```

### 4.2 Active Flows

```
# Set of active flow IDs per device
device:{device_id}:flows:active
    SET { "uuid1", "uuid2", "uuid3" }

# Flow details for active flows
flow:{flow_id}
    HASH {
        device_id: "phone-1",
        src_ip: "192.168.1.105",
        dst_ip: "142.250.80.46",
        dst_port: "443",
        app: "chrome",
        bytes_in: "12345",
        bytes_out: "6789",
        start_time: "2026-04-09T10:30:00Z"
    }
    TTL: 300s (cleaned up if flow not updated)
```

### 4.3 Live Bandwidth

```
# Per-device bandwidth (updated every second)
device:{device_id}:bandwidth
    HASH {
        bytes_in_sec: "1024",
        bytes_out_sec: "2048",
        packets_sec: "15"
    }
    TTL: 10s

# Global bandwidth (all devices)
stats:bandwidth:global
    HASH {
        total_bytes_in_sec: "3072",
        total_bytes_out_sec: "6144",
        device_count: "3"
    }
    TTL: 10s
```

### 4.4 Threat State

```
# Active threats sorted by severity (score: 1=low, 2=med, 3=high, 4=critical)
threats:active
    SORTED_SET {
        "threat:42": 3,
        "threat:43": 4,
        "threat:44": 1
    }

# Threat details
threat:{id}:detail
    HASH { ... threat fields ... }
    TTL: 3600s (1 hour cache, DB is source of truth)
```

### 4.5 Pub/Sub Channels

```
# Real-time event channels for WebSocket fan-out
channel:traffic:{device_id}     — per-device traffic events
channel:traffic:all             — all traffic events
channel:threats                 — new threat alerts
channel:stats                   — bandwidth stat updates
channel:device_status           — device online/offline changes
```

---

## 5. Continuous Aggregates (TimescaleDB)

Pre-computed materialized views for dashboard performance:

```sql
-- Hourly bandwidth per device
CREATE MATERIALIZED VIEW bandwidth_hourly
WITH (timescaledb.continuous) AS
SELECT
    device_id,
    time_bucket('1 hour', start_time) AS hour,
    SUM(total_bytes_in) AS bytes_in,
    SUM(total_bytes_out) AS bytes_out,
    COUNT(*) AS flow_count,
    COUNT(DISTINCT dst_ip) AS unique_destinations,
    COUNT(DISTINCT app_name) AS unique_apps
FROM flows
GROUP BY device_id, hour;

-- Hourly app usage per device
CREATE MATERIALIZED VIEW app_usage_hourly
WITH (timescaledb.continuous) AS
SELECT
    device_id,
    app_name,
    app_category,
    time_bucket('1 hour', start_time) AS hour,
    SUM(total_bytes_in + total_bytes_out) AS total_bytes,
    COUNT(*) AS flow_count
FROM flows
WHERE app_name IS NOT NULL
GROUP BY device_id, app_name, app_category, hour;

-- Daily threat summary
CREATE MATERIALIZED VIEW threats_daily
WITH (timescaledb.continuous) AS
SELECT
    device_id,
    severity,
    category,
    time_bucket('1 day', detected_at) AS day,
    COUNT(*) AS threat_count
FROM threats
GROUP BY device_id, severity, category, day;

-- Top destinations per day
CREATE MATERIALIZED VIEW top_destinations_daily
WITH (timescaledb.continuous) AS
SELECT
    device_id,
    dst_ip,
    dns_name,
    country_code,
    time_bucket('1 day', start_time) AS day,
    SUM(total_bytes_in + total_bytes_out) AS total_bytes,
    COUNT(*) AS connection_count
FROM flows
GROUP BY device_id, dst_ip, dns_name, country_code, day;
```

---

## 6. Data Retention Policy

| Data | Retention | Compression | Rationale |
|------|-----------|-------------|-----------|
| Raw packets | 7 days | After 1 day | High volume, needed only for forensics |
| Flows | 90 days | After 7 days | Core analytical data |
| Bandwidth stats | 30 days | After 3 days | Aggregated from flows |
| Continuous aggregates | 1 year | N/A | Pre-computed, small footprint |
| Threats | Indefinite | N/A | Security audit trail |
| Threat rules | Indefinite | N/A | Configuration data |
| Block/whitelist | Indefinite | N/A | Configuration data |
| Forensic dumps (.pcap) | 30 days | Compressed on save | Disk-intensive |

---

## 7. Migration Strategy

Migrations managed via **Alembic** (SQLAlchemy migration tool):

```
core/migrations/
├── alembic.ini
├── env.py
├── versions/
│   ├── 001_initial_schema.py
│   ├── 002_add_hypertables.py
│   ├── 003_add_retention_policies.py
│   ├── 004_add_continuous_aggregates.py
│   └── 005_add_blocklist_whitelist.py
└── script.py.mako
```

Initial setup:
```bash
# Applied automatically by docker-compose on first run
alembic upgrade head
```
