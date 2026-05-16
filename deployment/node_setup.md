# BGP-X Node Deployment Guide

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

This document is the step-by-step deployment guide for BGP-X relay, exit, domain bridge, and mesh nodes. It covers everything from initial server and hardware selection through to a running, monitored BGP-X node.

---

## 1. Prerequisites

### 1.1 Platform Requirements

| Platform | Requirements | Notes |
|---|---|---|
| Linux (x86_64) | Ubuntu 22.04+, Debian 12+, or equivalent | Primary server platform |
| Linux (ARM64) | Ubuntu 22.04+ arm64 (Raspberry Pi, Orange Pi, GL.iNet) | Edge and community deployment |
| OpenWrt | OpenWrt 23.05+ on MT7981B or newer | BGP-X Router v1, Node v1, Gateway v1 |
| Embedded (mesh-only) | bgpx-mesh-firmware package | ESP32-S3/nRF52840 devices (Tier 2 Client Node) |

### 1.2 Memory Requirements

| Node Role | Minimum RAM | Recommended RAM |
|---|---|---|
| Relay node | 256 MB | 512 MB |
| Exit node | 512 MB | 1 GB |
| Domain bridge node | 512 MB | 1 GB |
| Mesh island gateway | 512 MB | 1 GB |
| Full mesh node (radio + WAN) | 256 MB | 512 MB |

---

## 2. Server and Hardware Selection

### 2.1 Hosting Provider Considerations (Clearnet Nodes)

Choose a hosting provider based on:

| Factor | Recommendation |
|---|---|
| Network quality | Tier 1 or well-connected Tier 2 provider |
| Jurisdiction | Match your desired exit jurisdiction |
| BGP diversity | Prefer providers with their own ASN |
| Terms of service | Verify relay/exit nodes are permitted |
| DDoS protection | At minimum 1 Gbps mitigation |
| Payment | Privacy-respecting payment methods preferred |

Do NOT use providers with terms of service that prohibit:
- Running "proxy" or "anonymization" services
- High-volume traffic
- Operating a "server" on residential plans

### 2.2 Recommended Specifications (Server/VPS)

**Relay Node**: 4 cores, 2 GB RAM, 20 GB SSD, 1 Gbps unmetered
**Exit Gateway**: 8 cores, 8 GB RAM, 40 GB SSD, 1 Gbps unmetered
**Domain Bridge**: 4 cores, 4 GB RAM, 40 GB SSD, 1 Gbps unmetered (must have clearnet WAN + mesh radio capability)

### 2.3 Physical Hardware Selection (Router/Node/Gateway)

BGP-X native hardware is designed for specific deployment scenarios. Select the appropriate hardware for your intended role:

| Hardware Product | Target Role | Key Characteristics |
|---|---|---|
| BGP-X Router v1 | End-user AIO, replaces home router | Integrated LoRa/WiFi/BLE, WAN + LAN, desktop or outdoor enclosure |
| BGP-X Node v1 | Community relay, range extension | Outdoor IP67, solar/battery native, optional WAN |
| BGP-X Gateway v1 | Provider infrastructure | High throughput, SFP+ uplink, rack-mount |

For domain bridge nodes: requires hardware with **both** clearnet WAN interface and mesh radio (WiFi mesh or LoRa). BGP-X Node v1 with WAN option or BGP-X Router v1 are suitable.

See `/hardware/README.md` for the full hardware ecosystem details.

---

## 3. Initial Server Setup

### 3.1 System Updates and Dependencies

```bash
# Update system
apt-get update && apt-get upgrade -y

# Install required packages
apt-get install -y \
    curl \
    wget \
    gnupg \
    ufw \
    fail2ban \
    logrotate \
    prometheus-node-exporter \
    chrony  # NTP synchronization (critical for timestamp validation)

# Configure NTP (CRITICAL for timestamp validation)
systemctl enable chrony
systemctl start chrony
chronyc tracking  # Verify clock is synchronized
```

### 3.2 User and Directory Setup

```bash
# Create bgpx system user
useradd --system \
        --no-create-home \
        --shell /sbin/nologin \
        --comment "BGP-X Node Daemon" \
        bgpx

# Create directories
mkdir -p /etc/bgpx
mkdir -p /var/lib/bgpx
mkdir -p /var/run/bgpx
mkdir -p /var/log/bgpx

# Set ownership
chown -R bgpx:bgpx /etc/bgpx /var/lib/bgpx /var/run/bgpx /var/log/bgpx
chmod 700 /etc/bgpx
chmod 750 /var/lib/bgpx
```

### 3.3 Firewall Configuration

```bash
# Configure UFW
ufw default deny incoming
ufw default allow outgoing

# Allow SSH (change port if you've moved SSH)
ufw allow 22/tcp

# Allow BGP-X overlay traffic
ufw allow 7474/udp

# If running exit node: allow DNS lookups (UDP 53 outbound)
# This is already allowed by 'default allow outgoing'

# If running Prometheus monitoring (restrict to monitoring network)
ufw allow from 10.0.0.0/8 to any port 9474

# Enable firewall
ufw enable
ufw status verbose
```

### 3.4 System Performance Tuning

```bash
cat >> /etc/sysctl.d/99-bgpx.conf << EOF
# Network performance for BGP-X relay
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.rmem_default = 4194304
net.core.wmem_default = 4194304
net.core.netdev_max_backlog = 65536
net.core.somaxconn = 65536
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192

# Security hardening
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
EOF

sysctl -p /etc/sysctl.d/99-bgpx.conf
```

---

## 4. BGP-X Installation

### 4.1 Install from Package Repository

```bash
# Add BGP-X package repository
curl -fsSL https://pkg.bgpx.network/gpg.key \
    | gpg --dearmor \
    | tee /usr/share/keyrings/bgpx-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/bgpx-keyring.gpg] \
    https://pkg.bgpx.network/apt stable main" \
    | tee /etc/apt/sources.list.d/bgpx.list

apt-get update
apt-get install -y bgpx-node bgpx-cli
```

### 4.2 Install from Source (Development/Auditing)

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Clone repository
git clone https://github.com/bgpx-network/bgpx
cd bgpx

# Verify git commit signature
git verify-commit HEAD

# Build
cargo build --release --bin bgpx-node --bin bgpx-cli

# Install
install -m 755 target/release/bgpx-node /usr/local/bin/
install -m 755 target/release/bgpx-cli /usr/local/bin/
```

### 4.3 OpenWrt Installation

For BGP-X Router v1, Node v1, Gateway v1, or compatible third-party routers:

```bash
# Install on OpenWrt
opkg update && opkg install bgpx-node

# Configuration at /etc/bgpx/node.toml
# Data directory at /var/lib/bgpx (tmpfs — use /overlay for persistence)

# Enable service
/etc/init.d/bgpx-node enable
/etc/init.d/bgpx-node start
```

---

## 5. Key Generation and Identity

### 5.1 Generate Node Identity

```bash
# Generate the node private key (do this ONCE; back it up immediately)
bgpx-cli keygen --output /etc/bgpx/node_private_key

# Secure the key file
chown bgpx:bgpx /etc/bgpx/node_private_key
chmod 600 /etc/bgpx/node_private_key

# Display the NodeID (save this for your records)
bgpx-cli key-info --key /etc/bgpx/node_private_key
# Output:
# NodeID:     a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
# Public Key: base64url(...)
# Created:    2026-04-24T12:00:00Z

# IMMEDIATELY back up the private key
# Store a copy in at minimum 2 secure locations
# (encrypted password manager, offline encrypted storage, etc.)

# Example encrypted backup with GPG
gpg --symmetric --cipher-algo AES256 \
    /etc/bgpx/node_private_key
```

**CRITICAL**: The private key is the ONLY file that cannot be regenerated. If you lose it, your NodeID is permanently gone. Reputation built under the old NodeID is not transferable.

---

## 6. Configuration

BGP-X uses a single TOML configuration file at `/etc/bgpx/node.toml`.

### 6.1 Basic Configuration (Clearnet Relay Node)

```bash
# Detect public IP
PUBLIC_IP=$(curl -fsSL https://api.ipify.org)

cat > /etc/bgpx/node.toml << EOF
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
name = "my-bgpx-relay-1"
data_dir = "/var/lib/bgpx"
log_level = "info"
log_format = "json"
log_destination = "/var/log/bgpx/node.log"

[[routing_domains]]
domain_type = "clearnet"
domain_id = "0x00000001-00000000"
enabled = true
endpoints = [
    { protocol = "udp", address = "${PUBLIC_IP}", port = 7474 }
]

[advertisement]
bandwidth_mbps = 100
latency_ms = 15
region = "EU"
country = "DE"  # OPTIONAL — jurisdiction for geo plausibility scoring
asn = 12345

[sessions]
max_sessions = 10000
session_idle_timeout_seconds = 90
keepalive_interval_seconds = 25

[reputation]
geo_plausibility_enabled = true
enforce_global_blacklist = false  # Local blacklist is authoritative

[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"

[control_api]
socket_path = "/var/run/bgpx/control.sock"

[extensions]
cover_traffic = false
multiplexing = true
path_quality_reporting = true
pluggable_transport = false
mesh_transport = false
EOF

chown bgpx:bgpx /etc/bgpx/node.toml
chmod 640 /etc/bgpx/node.toml
```

**Note on `country` field**: This declares your node's jurisdiction. Geographic plausibility scoring (RTT-based verification) applies ONLY if you declare a jurisdiction. If you omit this field, geo plausibility scoring does NOT apply. Declaring jurisdiction is OPTIONAL.

**Note on satellite WAN**: If your WAN connection is via satellite (Starlink, Iridium, etc.), configure `latency_class` appropriately. Satellite internet is treated as clearnet domain (0x00000001), NOT a separate domain. Configure as:
```toml
[advertisement]
latency_class = "satellite-leo"  # or "satellite-geo"
```
Satellite-connected nodes are exempt from geo plausibility RTT scoring.

### 6.2 Exit Node Configuration

Add exit capability to a clearnet node:

```bash
cat >> /etc/bgpx/node.toml << 'EOF'

[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "100.64.0.0/10",
    "192.0.0.0/24",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "DE"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
max_outbound_connections = 2000
EOF

# Add "exit" to roles
sed -i 's/roles = \["relay", "entry", "discovery"\]/roles = ["relay", "entry", "exit", "discovery"]/' \
    /etc/bgpx/node.toml
```

**ECH Support**: `ech_capable = true` enables Encrypted Client Hello at the exit. Exit nodes will attempt ECH when the destination publishes ECH configuration in DNS HTTPS records. ECH hides the domain name from SNI, providing additional privacy.

### 6.3 Domain Bridge Node Configuration

Domain bridge nodes have endpoints in multiple routing domains (e.g., clearnet WAN + LoRa mesh). They enable cross-domain routing.

```bash
cat >> /etc/bgpx/node.toml << 'EOF'

[[routing_domains]]
domain_type = "mesh"
domain_id = "0x00000003-a1b2c3d4"
island_id = "lima-district-1"
enabled = true
mesh_transport = [
    { transport = "wifi_mesh", interface = "mesh0" },
    { transport = "lora", device = "/dev/ttyUSB0", frequency_mhz = 868.0, spreading_factor = 9 }
]

[domain_bridge]
enabled = true
bridges = [
    {
        from_domain = "0x00000001-00000000",
        to_domain = "0x00000003-a1b2c3d4",
        bridge_latency_ms = 25,
        bridge_transport = "wifi_mesh",
        available = true
    }
]

[mesh_island]
auto_publish_advertisement = true
advertisement_interval_hours = 8
EOF

# Add "domain_bridge" to roles
sed -i 's/roles = \["relay", "entry", "discovery"\]/roles = ["relay", "entry", "discovery", "domain_bridge"]/' \
    /etc/bgpx/node.toml
```

Domain bridge nodes run two concurrent path_id tables (single-domain and cross-domain). Ensure sufficient RAM (minimum 1 GB recommended; 2 GB for high-traffic production bridges).

### 6.4 Pool Membership Configuration

If your node is a member of named pools:

```bash
cat >> /etc/bgpx/node.toml << 'EOF'

[pools]
member_pools = ["pool-id-hex-1"]
discover_pools = ["bgpx-default"]
fallback_to_default = true
EOF
```

### 6.5 Validate Configuration

```bash
bgpx-node --config /etc/bgpx/node.toml --config-check
# Expected output:
# Configuration valid.
# NodeID: a3f2...
# Roles: relay, entry, discovery
# Routing Domains: 1 active
# Listen: 0.0.0.0:7474
# Public: <your_ip>:7474
```

---

## 7. systemd Service Installation

```bash
cat > /etc/systemd/system/bgpx-node.service << 'EOF'
[Unit]
Description=BGP-X Node Daemon
Documentation=https://docs.bgpx.network
After=network-online.target
Wants=network-online.target
Requires=time-sync.target

[Service]
Type=notify
User=bgpx
Group=bgpx
ExecStart=/usr/bin/bgpx-node --config /etc/bgpx/node.toml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10
TimeoutStopSec=60

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/bgpx /var/run/bgpx /var/log/bgpx
CapabilityBoundingSet=
AmbientCapabilities=
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
MemoryDenyWriteExecute=true
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
LockPersonality=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
PrivateUsers=true

# Resource limits
LimitNOFILE=65536
LimitNPROC=1024

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=bgpx-node

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable bgpx-node
systemctl start bgpx-node

# Verify it started correctly
systemctl status bgpx-node
journalctl -u bgpx-node -f --no-pager
```

---

## 8. Log Rotation

```bash
cat > /etc/logrotate.d/bgpx-node << 'EOF'
/var/log/bgpx/node.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        systemctl kill --signal=HUP bgpx-node || true
    endscript
}
EOF
```

---

## 9. Post-Deployment Verification

### 9.1 Basic Node Status

```bash
# Check node status
bgpx-cli status
# Expected:
# State:          ACTIVE
# NodeID:         a3f2...
# Roles:          relay, entry, discovery
# Active sessions: 0
# Uptime:         2m 15s
```

### 9.2 DHT Participation

```bash
# Check DHT connectivity
bgpx-cli dht stats
# Expected:
# Routing table: 150+ nodes
# Bootstrap nodes reachable: 5/5
# Advertisement: published, expires in 23h 45m

# Verify advertisement retrievable
bgpx-cli advertisement show
```

### 9.3 Domain Bridge Verification (if applicable)

```bash
# List configured domains
bgpx-cli domains list

# Verify bridge pair reachability
bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1"

# Test cross-domain connectivity
bgpx-cli domains test --from clearnet --to "mesh:lima-district-1"
```

### 9.4 Exit Node Verification (if applicable)

```bash
# Verify exit policy loaded
bgpx-cli exit-policy show

# Test ECH capability
bgpx-cli exit test-ech --destination example.com
```

### 9.5 Pool Membership Verification (if applicable)

```bash
# Verify pool membership confirmed
bgpx-cli pools verify
```

### 9.6 Network Reachability

```bash
# Check port is reachable (run from another machine)
nc -u -z <your_public_ip> 7474 && echo "Port reachable" || echo "Port NOT reachable"
```

---

## 10. Prometheus Monitoring

```bash
# Add metrics configuration
cat >> /etc/bgpx/node.toml << 'EOF'

[metrics]
prometheus_enabled = true
prometheus_addr = "127.0.0.1"
prometheus_port = 9474
EOF

systemctl reload bgpx-node

# Verify metrics endpoint
curl -s http://127.0.0.1:9474/metrics | head -20
```

---

## 11. Satellite WAN Configuration

For nodes connected to the internet via satellite (Starlink, Iridium, Inmarsat, HughesNet, Viasat):

**Important**: Satellite internet services provide BGP-routed IP connectivity. From BGP-X's perspective, they are **clearnet domain** (0x00000001), NOT a separate domain.

```toml
[advertisement]
# Satellite nodes use clearnet domain_id (default)
latency_class = "satellite-leo"  # Options: "satellite-leo", "satellite-meo", "satellite-geo"

# Satellite nodes are exempt from geo plausibility RTT scoring
# because terminal IP may be distant from ground station
```

Auto-detection: BGP-X daemon detects USB satellite modems (Starlink Gen 3, Iridium Certus) via USB vendor ID and applies satellite latency class automatically.

---

## 12. Backup Procedure

**Critical files to back up** (before any system changes):

```bash
# Node private key (MOST IMPORTANT)
# Back up immediately after key generation

# Configuration
cp /etc/bgpx/node.toml /secure-backup/bgpx-config-$(hostname)-$(date +%Y%m%d).toml

# Reputation database (optional, recoverable from network)
cp /var/lib/bgpx/reputation.db /secure-backup/bgpx-reputation-$(hostname).db
```

**If you lose the node private key**: your NodeID is permanently gone. You must generate a new key, which creates a new NodeID. Reputation built under the old NodeID is not transferable.

---

## 13. Post-Deployment Checklist

- [ ] `bgpx-cli status` shows ACTIVE
- [ ] DHT routing table has >20 entries
- [ ] Advertisement published (confirmed via `bgpx-cli advertisement show`)
- [ ] Node ID and public key documented and backed up
- [ ] Private key backed up to separate offline storage
- [ ] Firewall verified (UDP 7474 open; other ports closed)
- [ ] Log rotation configured and tested
- [ ] Prometheus metrics accessible from monitoring host
- [ ] If exit node: ECH test passes; exit policy reviewed; operator contact set
- [ ] If domain bridge: both domains reachable; bridge pair verified
- [ ] If pool member: pool membership confirmed via `bgpx-cli pools verify`
- [ ] NTP synchronization verified (`chronyc tracking`)
- [ ] systemd service enabled (starts on boot)

---

## 14. Ongoing Operations

### 14.1 Regular Maintenance Commands

```bash
# Check node health
bgpx-cli status

# Check statistics (last 24 hours)
bgpx-cli stats --window 24h

# View recent log entries (no sensitive data)
journalctl -u bgpx-node --since "1 hour ago"

# Force advertisement re-publication
bgpx-cli advertisement republish

# Update software
apt-get update && apt-get upgrade bgpx-node bgpx-cli
systemctl restart bgpx-node
```

### 14.2 Cross-Domain Operations

```bash
# Monitor bridge pair status
bgpx-cli domains bridges --self

# Check mesh island status (if gateway)
bgpx-cli domains island-status --island-id your-island-name

# Test cross-domain path
bgpx-cli paths build --domain-segments "clearnet:2,bridge:clearnet→mesh:your-island,mesh:your-island:2"
```

### 14.3 Secure Shutdown

Before shutting down or retiring a node:

```bash
# Publish NODE_WITHDRAW to gracefully leave network
bgpx-cli node withdraw

# Wait for propagation (up to 30 seconds)
sleep 30

# Stop service
systemctl stop bgpx-node
```

This allows active sessions to drain and informs other nodes via the DHT that this node is no longer available.
