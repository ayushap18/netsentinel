# NetSentinel — Setup Guide

**Date:** 2026-04-09

---

## Prerequisites

### Core Server (runs on your laptop or a dedicated machine)
- Docker & Docker Compose v2+
- 4GB RAM minimum (8GB recommended)
- 10GB free disk space
- Linux, macOS, or WSL2

### Laptop Agent
- Python 3.12+
- Root/sudo access (for packet capture)
- Linux or macOS

### Android Agent
- Termux app installed (from F-Droid, NOT Play Store)
- Python 3.12+ installed in Termux
- Root access (recommended) OR willingness to use VPN-TUN mode

### Optional
- Telegram account (for alerts)
- MaxMind account (for GeoIP — free registration)

---

## Step 1: Clone & Configure

```bash
# Clone the repository
git clone https://github.com/ayush/netsentinel.git
cd netsentinel

# Copy environment template
cp .env.example .env
```

Edit `.env` with your settings:
```bash
# Generate secrets (run these commands, paste output into .env)
python3 -c "import secrets; print(secrets.token_hex(32))"   # JWT_SECRET
python3 -c "import secrets; print(secrets.token_hex(16))"   # DB password
python3 -c "import secrets; print(secrets.token_hex(16))"   # Redis password

# Telegram (optional — get from @BotFather on Telegram)
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_ALLOWED_CHAT_IDS=your_chat_id
```

---

## Step 2: Start Core Services

```bash
# Start everything
docker-compose up -d

# Verify all services are healthy
docker-compose ps

# Expected output:
# netsentinel-core        running (healthy)    0.0.0.0:8000->8000
# netsentinel-dashboard   running (healthy)    0.0.0.0:3000->3000
# netsentinel-timescaledb running (healthy)    5432
# netsentinel-redis       running (healthy)    6379
# netsentinel-bot         running (healthy)    8001

# Check Core health
curl http://localhost:8000/api/health
```

---

## Step 3: Set Up Admin Account

```bash
# Run first-time setup wizard
docker exec -it netsentinel-core python -m netsentinel.setup

# This will:
# 1. Run database migrations
# 2. Create admin account (you set username/password)
# 3. Generate your first agent API key
# 4. Download threat feeds
# 5. Load default detection rules
```

---

## Step 4: Install Laptop Agent

```bash
# Install agent package
cd agent
pip install -e .

# Configure agent
netsen-agent setup \
    --server wss://localhost:8000/ws/agent \
    --name "my-laptop" \
    --key <api_key_from_step_3>

# Test connection
netsen-agent test

# Start agent (foreground for testing)
netsen-agent start

# Start agent (background service)
netsen-agent install-service   # Creates systemd/launchd service
netsen-agent start --daemon
```

### Verify Laptop Agent
```bash
# Check Core sees the device
curl -H "Authorization: Bearer <your_jwt>" http://localhost:8000/api/devices

# Should show your laptop as "online"
```

---

## Step 5: Install Android Agent (Termux)

### Option A: Rooted Phone (Recommended)

```bash
# In Termux:
pkg update && pkg install python root-repo tcpdump

# Install agent
pip install netsentinel-agent

# Configure
netsen-agent setup \
    --server wss://YOUR_LAPTOP_IP:8000/ws/agent \
    --name "my-phone" \
    --key <new_api_key> \
    --capture-mode tcpdump

# Start
netsen-agent start
```

### Option B: Non-Rooted Phone (VPN-TUN Mode)

```bash
# In Termux:
pkg update && pkg install python

# Install agent
pip install netsentinel-agent

# Configure with VPN mode
netsen-agent setup \
    --server wss://YOUR_LAPTOP_IP:8000/ws/agent \
    --name "my-phone" \
    --key <new_api_key> \
    --capture-mode vpn-tun

# Start (will prompt to allow VPN)
netsen-agent start
```

**Note:** VPN-TUN mode creates a local VPN on the phone to intercept traffic. You'll see a VPN icon in the status bar. All traffic stays local — nothing is routed externally.

### Keep Agent Running (Termux)
```bash
# Prevent Termux from being killed
termux-wake-lock

# Run agent in background
nohup netsen-agent start > ~/netsentinel.log 2>&1 &

# Or use Termux:Boot app to auto-start on phone boot
```

---

## Step 6: Install CLI Tool

```bash
cd cli
pip install -e .

# Configure CLI
netsen config set --server http://localhost:8000 --key <your_jwt_or_api_key>

# Test
netsen devices
netsen monitor
```

---

## Step 7: Set Up Telegram Alerts (Optional)

1. Message @BotFather on Telegram → `/newbot` → get bot token
2. Message your new bot → get your chat ID via `https://api.telegram.org/bot<TOKEN>/getUpdates`
3. Add to `.env`:
   ```
   TELEGRAM_BOT_TOKEN=<your_token>
   TELEGRAM_ALLOWED_CHAT_IDS=<your_chat_id>
   ```
4. Restart bot: `docker-compose restart bot`
5. Test: send `/status` to your bot

---

## Step 8: Set Up GeoIP (Optional)

1. Create free account at https://www.maxmind.com/en/geolite2/signup
2. Generate license key
3. Add to `.env`:
   ```
   MAXMIND_LICENSE_KEY=<your_key>
   ```
4. Download database:
   ```bash
   docker exec netsentinel-core python -m netsentinel.geoip_update
   ```

---

## Step 9: Verify Everything

```bash
# 1. Check all services
docker-compose ps

# 2. Check devices
netsen devices

# 3. Open dashboard
open http://localhost:3000

# 4. Generate some traffic on your phone/laptop

# 5. Check flows appear
netsen flows my-laptop --last 5m

# 6. Check threats (should be clean)
netsen threats

# 7. Test Telegram
# Send /status to your bot
```

---

## Troubleshooting

### Agent can't connect to Core
```bash
# Check Core is running
curl http://YOUR_IP:8000/api/health

# Check firewall allows port 8000
sudo ufw status  # Linux
sudo pfctl -sr   # macOS

# Check agent config
cat ~/.netsentinel/agent.conf
```

### No packets captured (laptop)
```bash
# Check permissions
sudo setcap cap_net_raw+eip $(which python3)
# OR run agent with sudo
sudo netsen-agent start
```

### No packets captured (Android)
```bash
# Rooted: check tcpdump
which tcpdump
su -c "tcpdump -c 5 -i any"

# Non-rooted: check VPN permission
# Android should have prompted for VPN — check notification shade
```

### Dashboard not loading
```bash
# Check dashboard container
docker logs netsentinel-dashboard

# Rebuild if needed
docker-compose build dashboard
docker-compose up -d dashboard
```

### High memory usage
```bash
# Check TimescaleDB
docker exec netsentinel-timescaledb psql -U netsentinel -c "SELECT pg_size_pretty(pg_database_size('netsentinel'));"

# Force compression
docker exec netsentinel-core python -m netsentinel.maintenance compress

# Check retention policies
docker exec netsentinel-timescaledb psql -U netsentinel -c "SELECT * FROM timescaledb_information.jobs WHERE proc_name = 'policy_retention';"
```
