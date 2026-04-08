# NetSentinel — Android Agent Guide

**Date:** 2026-04-09

Complete guide for setting up and running NetSentinel agent on Android devices via Termux.

---

## Overview

The Android agent runs inside Termux — a terminal emulator for Android that provides a Linux-like environment. It captures network traffic from your phone and streams it to the NetSentinel Core server.

**Two capture modes:**
| Mode | Root Required | Capture Depth | Battery Impact |
|------|--------------|---------------|----------------|
| **tcpdump** (recommended) | Yes | Full packet headers | Low |
| **VPN-TUN** | No | Full packet headers | Medium |

---

## Prerequisites

### Install Termux
**Important:** Install from F-Droid, NOT Google Play Store (Play Store version is outdated).

1. Install F-Droid: https://f-droid.org/
2. Search "Termux" in F-Droid → Install
3. Open Termux

### Initial Termux Setup
```bash
# Update packages
pkg update && pkg upgrade -y

# Install essentials
pkg install python openssh -y

# For rooted phones, also install:
pkg install root-repo tcpdump -y
```

---

## Installation

```bash
# Install NetSentinel agent
pip install netsentinel-agent

# Verify installation
netsen-agent --version
```

---

## Configuration

### Find Your Core Server IP
On the machine running NetSentinel Core:
```bash
# Linux
hostname -I | awk '{print $1}'

# macOS
ipconfig getifaddr en0
```

### Setup Agent
```bash
# Interactive setup
netsen-agent setup

# Or one-line setup
netsen-agent setup \
    --server wss://192.168.1.100:8000/ws/agent \
    --name "ayush-phone" \
    --key ns_ak_your_api_key_here \
    --capture-mode tcpdump    # or vpn-tun
```

This creates `~/.netsentinel/agent.conf`:
```yaml
server: wss://192.168.1.100:8000/ws/agent
device_name: ayush-phone
device_id: phone-ayush-abc123
api_key: ns_ak_...
capture_mode: tcpdump
battery_saver: true
battery_threshold: 20
log_level: info
```

---

## Capture Modes

### Mode 1: tcpdump (Rooted Phones)

Directly captures packets using tcpdump — the same tool used by network professionals.

```bash
# Test tcpdump access
su -c "tcpdump -c 10 -i any -n"

# If this works, you're good to go
netsen-agent setup --capture-mode tcpdump
```

**Pros:** Low overhead, reliable, deep packet access
**Cons:** Requires root

### Mode 2: VPN-TUN (No Root Required)

Creates a local VPN tunnel on the phone. All traffic passes through the tunnel where the agent captures it. Traffic is NOT routed externally — it goes straight back to the phone's network.

```bash
netsen-agent setup --capture-mode vpn-tun
```

When you start the agent, Android will prompt: "NetSentinel wants to set up a VPN connection." Tap "OK".

**Pros:** No root needed
**Cons:** Slightly higher battery usage, VPN icon in status bar, some apps may detect VPN

---

## Running the Agent

### Foreground (Testing)
```bash
netsen-agent start
# Ctrl+C to stop
```

### Background
```bash
# Start in background
nohup netsen-agent start > ~/netsentinel.log 2>&1 &

# Check if running
netsen-agent status

# Stop
netsen-agent stop
```

### Keep Termux Alive
Android aggressively kills background apps. To prevent this:

```bash
# 1. Acquire wake lock (keeps Termux running)
termux-wake-lock

# 2. Disable battery optimization for Termux
#    Settings → Apps → Termux → Battery → Unrestricted

# 3. Optional: Install Termux:Boot (from F-Droid) for auto-start
#    Create boot script:
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/netsentinel.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
termux-wake-lock
netsen-agent start --daemon
EOF
chmod +x ~/.termux/boot/netsentinel.sh
```

---

## Battery Management

The agent has built-in battery awareness:

```yaml
# In agent.conf
battery_saver: true          # Enable battery-aware mode
battery_threshold: 20        # Below this %, reduce capture
```

**Battery saver behavior:**
| Battery Level | Behavior |
|--------------|----------|
| Above threshold | Full capture, all packets processed |
| Below threshold | Reduced capture (sample 1 in 10 packets), longer heartbeat interval (60s) |
| Below 10% | Minimal mode: heartbeat only, no capture |
| Charging | Full capture regardless of level |

---

## Monitoring Agent Health

```bash
# Check status
netsen-agent status

# Output:
# Status: running (PID 12345)
# Connected: yes (Core at 192.168.1.100:8000)
# Uptime: 2h 15m
# Packets captured: 45,230
# Flows reported: 1,234
# Battery: 78% (full capture mode)
# Network: wifi (192.168.1.105)

# View logs
netsen-agent logs
netsen-agent logs --tail 50
netsen-agent logs --level error
```

---

## Troubleshooting

### "Permission denied" on tcpdump
```bash
# Check root access
su -c "whoami"  # Should print "root"

# If su doesn't work, your phone isn't rooted
# Switch to VPN-TUN mode:
netsen-agent setup --capture-mode vpn-tun
```

### Agent disconnects frequently
```bash
# Check if Termux is being killed
# Fix: disable battery optimization for Termux
# Settings → Apps → Termux → Battery → Unrestricted

# Also acquire wake lock
termux-wake-lock

# Check network stability
ping -c 10 192.168.1.100
```

### High battery drain
```bash
# Enable battery saver
netsen-agent config set battery_saver true
netsen-agent config set battery_threshold 30

# Reduce capture intensity
netsen-agent config set capture_sample_rate 0.5  # Capture 50% of packets
```

### VPN-TUN mode: Some apps don't work
Some apps detect VPN and refuse to work. Options:
1. Add those apps to the VPN bypass list:
   ```bash
   netsen-agent config add vpn_bypass com.example.app
   ```
2. Switch to tcpdump mode (requires root)

### Agent can't reach Core server
```bash
# Check connectivity
ping 192.168.1.100

# Check if Core WebSocket is accessible
python3 -c "
import asyncio, websockets
async def test():
    async with websockets.connect('ws://192.168.1.100:8000/ws/agent') as ws:
        print('Connected!')
asyncio.run(test())
"

# If on mobile data (not same WiFi):
# Core server must be accessible from internet
# Options: port forwarding, Tailscale, or Cloudflare Tunnel
```

### Connecting over mobile data (different network)
When your phone is on 4G/5G and Core is on home WiFi:

**Option 1: Tailscale (Recommended)**
```bash
# Install Tailscale on both Core machine and phone
# Both devices get a Tailscale IP (100.x.x.x)
# Use Tailscale IP as Core server address
netsen-agent config set server wss://100.64.0.1:8000/ws/agent
```

**Option 2: Port forwarding**
- Forward port 8000 on your router to your Core machine
- Use your public IP as server address
- **Security risk:** exposes Core to internet — use strong API keys

---

## Data Usage

The agent sends data to Core — this uses some of your mobile data plan.

**Estimated data usage:**
| Activity Level | Data Sent to Core / Hour |
|---------------|-------------------------|
| Light (browsing) | ~5-10 MB |
| Medium (social media) | ~15-30 MB |
| Heavy (streaming, gaming) | ~50-100 MB |

**To reduce data usage:**
```bash
# Increase local aggregation (send summaries, not raw packets)
netsen-agent config set aggregation_level high

# Reduce heartbeat frequency
netsen-agent config set heartbeat_interval 60

# Only send flagged packets (not all)
netsen-agent config set send_mode flags_only
```
