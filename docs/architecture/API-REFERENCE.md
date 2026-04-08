# NetSentinel — API Reference

**Version:** 1.0
**Base URL:** `http://localhost:8000/api`
**Auth:** Bearer JWT token (header: `Authorization: Bearer <token>`)

---

## 1. Authentication

### POST `/auth/login`
Authenticate and receive JWT tokens.

**Request:**
```json
{
    "username": "admin",
    "password": "your-password"
}
```

**Response (200):**
```json
{
    "access_token": "eyJhbG...",
    "refresh_token": "eyJhbG...",
    "token_type": "bearer",
    "expires_in": 3600
}
```

### POST `/auth/refresh`
Refresh an expired access token.

**Request:**
```json
{
    "refresh_token": "eyJhbG..."
}
```

### POST `/auth/agent-key`
Generate a new agent API key.

**Response (201):**
```json
{
    "agent_key": "ns_ak_abc123...",
    "created_at": "2026-04-09T10:00:00Z"
}
```

---

## 2. Devices

### GET `/devices`
List all registered devices.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| status | string | all | Filter: "online", "offline", "quarantined" |

**Response (200):**
```json
{
    "devices": [
        {
            "id": "phone-1",
            "name": "Ayush's Phone",
            "device_type": "android",
            "status": "online",
            "network_type": "wifi",
            "local_ip": "192.168.1.105",
            "public_ip": "203.0.113.42",
            "last_seen": "2026-04-09T10:30:00Z",
            "bandwidth": {
                "bytes_in_sec": 1024,
                "bytes_out_sec": 2048
            },
            "active_flows": 12,
            "active_threats": 0
        }
    ],
    "total": 3,
    "online": 2
}
```

### GET `/devices/{device_id}`
Get detailed device information.

**Response (200):**
```json
{
    "id": "phone-1",
    "name": "Ayush's Phone",
    "device_type": "android",
    "status": "online",
    "network_type": "wifi",
    "local_ip": "192.168.1.105",
    "public_ip": "203.0.113.42",
    "os_info": "Android 14",
    "agent_version": "0.1.0",
    "last_seen": "2026-04-09T10:30:00Z",
    "registered_at": "2026-04-01T08:00:00Z",
    "metadata": {
        "battery": 78,
        "cpu_percent": 12.5,
        "agent_uptime_sec": 3600
    },
    "stats": {
        "bandwidth": { "bytes_in_sec": 1024, "bytes_out_sec": 2048 },
        "active_flows": 12,
        "total_flows_24h": 1543,
        "total_bytes_24h": 524288000,
        "active_threats": 0,
        "total_threats_24h": 2
    }
}
```

### PUT `/devices/{device_id}`
Update device settings.

**Request:**
```json
{
    "name": "Ayush's Primary Phone",
    "metadata": { "location": "home" }
}
```

### DELETE `/devices/{device_id}`
Unregister a device (does not delete historical data).

### POST `/devices/{device_id}/quarantine`
Quarantine a device — blocks all non-essential traffic.

### POST `/devices/{device_id}/unquarantine`
Remove quarantine from a device.

---

## 3. Flows

### GET `/devices/{device_id}/flows`
List network flows for a device.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| status | string | all | "active", "completed", "blocked" |
| app | string | all | Filter by app name |
| category | string | all | Filter by app category |
| protocol | string | all | "TCP", "UDP", etc. |
| dst_ip | string | — | Filter by destination IP |
| dns_name | string | — | Filter by domain (partial match) |
| country | string | — | Filter by country code |
| since | datetime | 1h ago | Start time |
| until | datetime | now | End time |
| sort | string | start_time | Sort field |
| order | string | desc | "asc" or "desc" |
| limit | int | 50 | Max results (max 500) |
| offset | int | 0 | Pagination offset |

**Response (200):**
```json
{
    "flows": [
        {
            "id": "550e8400-e29b-41d4-a716-446655440000",
            "device_id": "phone-1",
            "start_time": "2026-04-09T10:25:00Z",
            "end_time": null,
            "src_ip": "192.168.1.105",
            "dst_ip": "157.240.1.35",
            "src_port": 54321,
            "dst_port": 443,
            "protocol": "TCP",
            "total_bytes_in": 125000,
            "total_bytes_out": 45000,
            "packet_count": 230,
            "avg_rtt_ms": 45.2,
            "app_name": "instagram",
            "app_category": "social",
            "dns_name": "scontent.cdninstagram.com",
            "tls_version": "TLS 1.3",
            "country_code": "US",
            "city": "Menlo Park",
            "as_org": "Facebook, Inc.",
            "status": "active"
        }
    ],
    "total": 342,
    "has_more": true
}
```

### GET `/flows/{flow_id}`
Get detailed flow information including packet summary.

### GET `/flows/{flow_id}/packets`
Get packets belonging to a specific flow.

---

## 4. Packets

### GET `/devices/{device_id}/packets`
Query packet captures for a device.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| protocol | string | all | "TCP", "UDP", "DNS", "ICMP" |
| src_ip | string | — | Filter by source IP |
| dst_ip | string | — | Filter by destination IP |
| port | int | — | Filter by src or dst port |
| since | datetime | 5m ago | Start time |
| until | datetime | now | End time |
| limit | int | 100 | Max results (max 1000) |

**Response (200):**
```json
{
    "packets": [
        {
            "id": 123456,
            "timestamp": "2026-04-09T10:30:01.234Z",
            "src_ip": "192.168.1.105",
            "dst_ip": "8.8.8.8",
            "src_port": 54321,
            "dst_port": 53,
            "protocol": "DNS",
            "payload_size": 64,
            "direction": "outbound",
            "flow_id": "550e8400-...",
            "raw_summary": {
                "dns_query": "api.instagram.com",
                "dns_type": "A"
            }
        }
    ],
    "total": 5432
}
```

### GET `/devices/{device_id}/packets/export`
Export packets as .pcap file.

**Query Parameters:** Same as above.

**Response:** Binary `.pcap` file download.

---

## 5. Threats

### GET `/threats`
List all threats across all devices.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| status | string | active | "active", "blocked", "dismissed", "whitelisted", "resolved" |
| severity | string | all | "low", "medium", "high", "critical" |
| category | string | all | Threat category filter |
| device_id | string | all | Filter by device |
| since | datetime | 24h ago | Start time |
| limit | int | 50 | Max results |

**Response (200):**
```json
{
    "threats": [
        {
            "id": 42,
            "device_id": "phone-1",
            "device_name": "Ayush's Phone",
            "detected_at": "2026-04-09T10:28:00Z",
            "severity": "high",
            "category": "malware_c2",
            "title": "Connection to known C2 server",
            "description": "Device phone-1 established TCP connection to 198.51.100.50:4444, which is listed in abuse.ch C2 feed.",
            "evidence": {
                "dst_ip": "198.51.100.50",
                "dst_port": 4444,
                "threat_feed": "abuse.ch",
                "flow_bytes": 1234,
                "flow_duration_sec": 5
            },
            "status": "blocked",
            "action_taken": "blocked",
            "rule_id": 1,
            "forensic_path": "/data/forensics/threat_42.pcap"
        }
    ],
    "total": 5,
    "by_severity": {
        "critical": 0,
        "high": 1,
        "medium": 2,
        "low": 2
    }
}
```

### GET `/threats/{threat_id}`
Get full threat details including forensic evidence.

### POST `/threats/{threat_id}/action`
Take action on a threat.

**Request:**
```json
{
    "action": "block",       // "block", "whitelist", "dismiss", "investigate"
    "note": "Confirmed malicious"
}
```

### POST `/threats/{threat_id}/resolve`
Mark a threat as resolved.

---

## 6. Rules

### GET `/rules`
List all threat detection rules.

### POST `/rules`
Create a new rule.

**Request:**
```json
{
    "name": "block_suspicious_port",
    "description": "Block connections to port 4444 (common reverse shell)",
    "category": "network",
    "severity": "high",
    "rule_type": "port_match",
    "condition": {
        "dst_port": 4444,
        "direction": "outbound"
    },
    "action": "block",
    "notify": true
}
```

### PUT `/rules/{rule_id}`
Update an existing rule.

### DELETE `/rules/{rule_id}`
Delete a rule.

### POST `/rules/import`
Import rules from YAML file.

### GET `/rules/export`
Export all rules as YAML.

---

## 7. Statistics

### GET `/stats/bandwidth`
Get bandwidth statistics over time.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| device_id | string | all | Filter by device |
| interval | string | 1h | "1m", "5m", "1h", "1d" |
| since | datetime | 24h ago | Start time |
| until | datetime | now | End time |

**Response (200):**
```json
{
    "data": [
        {
            "timestamp": "2026-04-09T10:00:00Z",
            "bytes_in": 52428800,
            "bytes_out": 10485760,
            "flow_count": 234,
            "device_breakdown": {
                "phone-1": { "bytes_in": 31457280, "bytes_out": 6291456 },
                "laptop": { "bytes_in": 20971520, "bytes_out": 4194304 }
            }
        }
    ]
}
```

### GET `/stats/top-talkers`
Get top destination IPs/domains by traffic volume.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| device_id | string | all | Filter by device |
| group_by | string | dns_name | "dst_ip", "dns_name", "country", "app" |
| since | datetime | 24h ago | Start time |
| limit | int | 20 | Max results |

### GET `/stats/apps`
Get per-app bandwidth breakdown.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| device_id | string | all | Filter by device |
| since | datetime | 24h ago | Start time |

**Response (200):**
```json
{
    "apps": [
        {
            "name": "instagram",
            "category": "social",
            "total_bytes": 104857600,
            "bytes_in": 94371840,
            "bytes_out": 10485760,
            "flow_count": 89,
            "percentage": 35.2
        }
    ]
}
```

### GET `/stats/protocols`
Get traffic breakdown by protocol.

### GET `/stats/geo`
Get geographic distribution of connections (for map visualization).

---

## 8. Blocklist & Whitelist

### GET `/blocklist`
List all blocked IPs/domains.

### POST `/blocklist`
Add an entry to the blocklist.

**Request:**
```json
{
    "entry_type": "ip",      // "ip", "domain", "cidr"
    "value": "198.51.100.50",
    "reason": "Known C2 server",
    "expires_at": null       // null = permanent
}
```

### DELETE `/blocklist/{id}`
Remove an entry from the blocklist.

### GET `/whitelist`
List all whitelisted IPs/domains.

### POST `/whitelist`
Add an entry to the whitelist.

### DELETE `/whitelist/{id}`
Remove an entry from the whitelist.

---

## 9. System

### GET `/health`
System health check.

**Response (200):**
```json
{
    "status": "healthy",
    "version": "0.1.0",
    "uptime_sec": 86400,
    "components": {
        "database": "healthy",
        "redis": "healthy",
        "agent_manager": "healthy",
        "pipeline": "healthy",
        "telegram_bot": "healthy"
    },
    "agents_connected": 2,
    "pipeline_lag_ms": 150
}
```

### GET `/health/pipeline`
Detailed pipeline health and performance metrics.

---

## 10. WebSocket Endpoints

All WebSocket connections require authentication via query parameter: `?token=<jwt_token>`

### `ws://localhost:8000/ws/live-traffic`
Real-time traffic stream (all devices).

**Messages sent (server → client):**
```json
{
    "type": "flow_update",
    "device_id": "phone-1",
    "flow": { "...flow object..." }
}
```

### `ws://localhost:8000/ws/device/{device_id}/packets`
Real-time packet stream for a specific device.

### `ws://localhost:8000/ws/threats`
Real-time threat alerts.

**Messages sent:**
```json
{
    "type": "new_threat",
    "threat": { "...threat object..." }
}
```

### `ws://localhost:8000/ws/stats`
Live statistics updates (1 Hz).

**Messages sent:**
```json
{
    "type": "stats_update",
    "global_bandwidth": { "bytes_in_sec": 3072, "bytes_out_sec": 6144 },
    "devices": {
        "phone-1": { "bytes_in_sec": 1024, "bytes_out_sec": 2048, "status": "online" },
        "laptop": { "bytes_in_sec": 2048, "bytes_out_sec": 4096, "status": "online" }
    }
}
```

---

## 11. Error Format

All errors follow a consistent format:

```json
{
    "error": {
        "code": "DEVICE_NOT_FOUND",
        "message": "Device with id 'phone-3' not found",
        "status": 404
    }
}
```

**Common error codes:**
| Code | HTTP Status | Description |
|------|-------------|-------------|
| UNAUTHORIZED | 401 | Missing or invalid token |
| FORBIDDEN | 403 | Insufficient permissions |
| DEVICE_NOT_FOUND | 404 | Device ID doesn't exist |
| FLOW_NOT_FOUND | 404 | Flow ID doesn't exist |
| THREAT_NOT_FOUND | 404 | Threat ID doesn't exist |
| RULE_NOT_FOUND | 404 | Rule ID doesn't exist |
| VALIDATION_ERROR | 422 | Invalid request body |
| RATE_LIMITED | 429 | Too many requests |
| INTERNAL_ERROR | 500 | Server error |
