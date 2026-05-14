# Installing BGP-X on a Standalone Device (Laptop, Desktop, Server)

**Version**: 0.1.0-draft

---

## 1. Overview

You can run the BGP-X daemon on any Linux device: laptop, desktop, server, or Raspberry Pi. This protects that device's traffic without requiring a dedicated BGP-X router. You can also host `.bgpx` services from a standalone device, or configure it as a relay or exit node for the BGP-X network.

**Deployment Mode**: This is **Mode 3 — Standalone Device** from the BGP-X deployment spectrum.

**What standalone deployment provides**:
- Traffic from the device is routed through the BGP-X overlay
- Full access to `.bgpx` native services
- Can participate as a relay node (forwarding traffic for others)
- Can operate as an exit node (if configured with exit policy)
- Can serve as a domain bridge node (with appropriate interfaces)
- No router modifications required

**What standalone deployment does NOT provide**:
- Protection for other devices on your network (use a BGP-X Router v1 for that)
- Transparent protection for all applications without per-device config (the daemon must be running on each device you want protected)

---

## 2. Hardware Options

### Fully Supported Platforms

| Platform | Architecture | Notes |
|---|---|---|
| **Laptop/Desktop** | x86_64 | Primary target for most users |
| **Server** | x86_64 | Recommended for relay/exit operation |
| **Raspberry Pi 4/5** | aarch64 (ARM64) | Full bgpx-node daemon supported |
| **Orange Pi** | aarch64 | Full support on ARM64 models |
| **Banana Pi BPi-R3** | aarch64 | Full support; excellent router platform |
| **GL.iNet Routers** | aarch64/mipsel | Full support via OpenWrt package |
| **x86 Mini PCs** | x86_64 | Full support (Intel NUC, Minisforum, etc.) |
| **VMs / VPS** | x86_64 | Full support; excellent for relay/exit nodes |

### Resource Requirements

| Role | CPU | RAM | Storage | Network |
|---|---|---|---|---|
| **Client only** | 2 cores | 2 GB | 1 GB | Any |
| **Relay node** | 2 cores | 4 GB | 4 GB | 100 Mbps+ |
| **Exit node** | 4 cores | 8 GB | 16 GB | 1 Gbps+ |
| **Domain bridge** | 4 cores | 4 GB | 8 GB | 2 interfaces |

### Hardware Recommendations by Use Case

| Use Case | Recommended Hardware | Notes |
|---|---|---|
| **Personal laptop** | Any modern laptop | Run as user service; battery-aware |
| **Home server** | Raspberry Pi 5 or x86 mini PC | Run as system service |
| **Community relay** | x86 server or high-end ARM | Public relay; 24/7 operation |
| **Exit node operator** | x86 server, 4+ cores, 1 Gbps WAN | Provider-grade; see production/sop.md |
| **Domain bridge** | Device with 2+ interfaces | Requires clearnet + mesh transport |
| **Development testing** | Any machine | VM or bare metal |

---

## 3. Installation

### 3.1 Ubuntu / Debian

```bash
# Add BGP-X package repository
curl -fsSL https://packages.bgpx.org/linux/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/bgpx.gpg
echo "deb [signed-by=/usr/share/keyrings/bgpx.gpg] https://packages.bgpx.org/linux stable main" \
    | sudo tee /etc/apt/sources.list.d/bgpx.list
sudo apt update

# Install BGP-X daemon and CLI
sudo apt install bgpx-node bgpx-cli

# Verify installation
bgpx-cli --version
bgpx-node --version
```

### 3.2 Fedora / RHEL / CentOS

```bash
# Add BGP-X RPM repository
sudo dnf config-manager --add-repo https://packages.bgpx.org/linux/fedora/bgpx.repo

# Install
sudo dnf install bgpx-node bgpx-cli
```

### 3.3 Raspberry Pi OS (Debian-based)

```bash
# Use the same Ubuntu/Debian instructions above
# Ensure you're running the 64-bit kernel (aarch64)
# Check: uname -m
# Should output: aarch64
```

### 3.4 Arch Linux

```bash
# From AUR
yay -S bgpx-node bgpx-cli

# Or manually
git clone https://aur.archlinux.org/bgpx-node.git
cd bgpx-node
makepkg -si
```

### 3.5 Other Linux (Binary Install)

```bash
# Download latest release
wget https://releases.bgpx.org/bgpx-0.1.0-linux-x86_64.tar.gz
wget https://releases.bgpx.org/bgpx-0.1.0-linux-x86_64.tar.gz.sig

# Verify signature
gpg --import /usr/share/bgpx/release_pub.pem 2>/dev/null || \
    curl -fsSL https://releases.bgpx.org/linux/key.gpg | gpg --import
gpg --verify bgpx-0.1.0-linux-x86_64.tar.gz.sig

# Extract and install
tar xzf bgpx-0.1.0-linux-x86_64.tar.gz
sudo cp bgpx-node bgpx-cli /usr/local/bin/
sudo mkdir -p /etc/bgpx /var/run/bgpx /var/lib/bgpx
sudo chmod 700 /etc/bgpx /var/lib/bgpx
```

### 3.6 macOS (Experimental)

```bash
# Via Homebrew (when available)
brew install bgpx-node bgpx-cli

# Or manual install from release
curl -fsSL https://releases.bgpx.org/bgpx-0.1.0-macos-arm64.tar.gz | tar xz
sudo mv bgpx-node bgpx-cli /usr/local/bin/
```

### 3.7 Windows (via WSL2)

```bash
# Install WSL2 with Ubuntu
wsl --install -d Ubuntu

# Inside WSL2 Ubuntu session, follow Ubuntu/Debian instructions above
```

---

## 4. Initial Setup

### 4.1 Generate Node Identity

```bash
# System-wide installation (requires root)
sudo bgpx-keygen
# Generates /etc/bgpx/node.key (Ed25519 private key)
# NodeID stored in /etc/bgpx/node_id

# User-only installation (no root required)
mkdir -p ~/.bgpx
bgpx-keygen --output ~/.bgpx/node.key
# NodeID stored in ~/.bgpx/node_id
```

**Important**: The node key is your node's long-term identity. Back it up securely. If compromised or lost, generate a new key — you will have a new NodeID.

### 4.2 Start the Daemon

**System-wide (root, all users protected)**:
```bash
# Enable and start via systemd
sudo systemctl enable bgpx-node
sudo systemctl start bgpx-node

# Check status
sudo systemctl status bgpx-node

# View logs
sudo journalctl -u bgpx-node -f
```

**User-only (no root, single user protected)**:
```bash
# Enable as user service
systemctl --user enable bgpx-node
systemctl --user start bgpx-node

# Check status
systemctl --user status bgpx-node

# View logs
journalctl --user -u bgpx-node -f
```

### 4.3 Verify Operation

```bash
# Check daemon is running
bgpx-cli node stats

# View your NodeID
bgpx-cli node identity

# Check DHT connectivity
bgpx-cli node dht-stats

# Test a simple path construction
bgpx-cli paths test --destination 8.8.8.8
```

---

## 5. Configuration

### 5.1 Default Configuration Location

- **System-wide**: `/etc/bgpx/config.toml`
- **User-only**: `~/.bgpx/config.toml`

### 5.2 Minimal Configuration (Client Mode)

This configuration routes all traffic through the BGP-X overlay:

```toml
# /etc/bgpx/config.toml

[node]
role = "relay"           # Participate as relay (optional)

[sessions]
max_sessions = 1000

[paths]
default_hops = 4
min_hops = 3

[pools]
default_pool = "bgpx-default"

[[routing_policy.rules]]
priority = 10
destination = "0.0.0.0/0"
action = "bgpx"
```

### 5.3 Configuration with Selective Bypass

Route most traffic through BGP-X, but bypass for specific destinations:

```toml
[[routing_policy.rules]]
priority = 1
destination = "192.168.0.0/16"
action = "standard"
reason = "local_network"

[[routing_policy.rules]]
priority = 2
destination = "10.0.0.0/8"
action = "standard"
reason = "vpn_work"

[[routing_policy.rules]]
priority = 100
destination = "0.0.0.0/0"
action = "bgpx"
```

### 5.4 Configuration with Cross-Domain Path

Route specific destinations through mesh islands:

```toml
# Route sensitive traffic via a trusted mesh island
[[routing_policy.rules]]
priority = 5
destination_domain = "*.ultra-sensitive.org"
action = "bgpx"

[routing_policy.rules.path_constraints.domain_segments]
segments = [
    { type = "segment", domain = "clearnet", hops = 2 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 2 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "eu-nolog-exits", hops = 1, exit = true }
]
require_ech = true
```

### 5.5 Full Configuration Reference

```toml
# /etc/bgpx/config.toml — Complete Example

[node]
key_path = "/etc/bgpx/node.key"
data_dir = "/var/lib/bgpx"
log_level = "info"           # error, warn, info (not debug/trace in production)
role = "relay"               # relay, exit, domain_bridge

[network]
listen_addr = "0.0.0.0"
listen_port = 7474
ipv6 = true

[tun]
interface = "bgpx0"
mtu = 1280

[sessions]
max_sessions = 1000
idle_timeout_secs = 90
session_rehandshake_age_hours = 24

[paths]
default_hops = 4
min_hops = 3
# No max_hops — N-hop unlimited

[pools]
default_pool = "bgpx-default"
fallback_to_default = true

[reputation]
enforce_global_blacklist = false
local_blacklist_expiry_days = 30
geo_plausibility_enabled = true

[cover_traffic]
enabled = false             # Disabled for laptop; saves bandwidth

[pluggable_transport]
enabled = false

[performance]
worker_threads = 2          # Adjust for your CPU

# Routing Policy
[[routing_policy.rules]]
priority = 1
destination = "192.168.0.0/16"
action = "standard"

[[routing_policy.rules]]
priority = 2
destination = "10.0.0.0/8"
action = "standard"

[[routing_policy.rules]]
priority = 100
destination = "0.0.0.0/0"
action = "bgpx"

[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"
# tcp_listen = ""           # Empty = disabled; or "127.0.0.1:7475" for TCP

[control_api]
socket_path = "/var/run/bgpx/control.sock"

[metrics]
enabled = true
prometheus_listen = "127.0.0.1:9090"
```

---

## 6. Connecting BGP-X Browser

BGP-X Browser is the primary way to access `.bgpx` services from a standalone device. It auto-discovers the local daemon using this priority order:

1. `$BGPX_SOCKET` environment variable
2. `/var/run/bgpx/sdk.sock` (system-wide installation)
3. `~/.bgpx/sdk.sock` (user-only installation)
4. `127.0.0.1:7475` (TCP fallback if configured)

### 6.1 System-wide Installation

BGP-X Browser connects automatically. No configuration needed.

### 6.2 User-only Installation

If running as a user service, ensure the socket is accessible:

```bash
# Check socket exists
ls -la ~/.bgpx/sdk.sock

# Set environment variable (optional, for explicit configuration)
export BGPX_SOCKET=~/.bgpx/sdk.sock

# Start BGP-X Browser
# It will auto-discover the socket
```

### 6.3 Manual Connection in BGP-X Browser

If auto-discovery fails:

1. Open BGP-X Browser
2. Settings → Connection → Advanced
3. Socket path: `/var/run/bgpx/sdk.sock` (system) or `~/.bgpx/sdk.sock` (user)
4. Or enable TCP endpoint: `127.0.0.1:7475`

### 6.4 TCP Endpoint for LAN Access

To allow other devices on your LAN to use your standalone daemon:

```toml
[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"
tcp_listen = "192.168.1.100:7475"  # Your LAN IP
tcp_auth_required = true
```

Other devices can then connect via:
```bash
export BGPX_SOCKET=192.168.1.100:7475
```

**Security note**: TCP endpoints require authentication. Generate an auth token:
```bash
bgpx-cli sdk generate-token --output ~/.bgpx/auth_token
```

---

## 7. Using bgpx-cli

The command-line client (`bgpx-cli`) is your primary interface for managing the BGP-X daemon.

### 7.1 Node Management

```bash
# View node status and statistics
bgpx-cli node stats

# View your NodeID
bgpx-cli node identity

# Check DHT connectivity
bgpx-cli node dht-stats

# View recent events
bgpx-cli node events --last 1h
```

### 7.2 Path Management

```bash
# List active paths
bgpx-cli paths list

# Test path to a destination
bgpx-cli paths test --destination example.com

# Build a specific path
bgpx-cli paths build --destination 8.8.8.8 --hops 5 --pool bgpx-default

# Rebuild a degraded path
bgpx-cli paths rebuild <path_id>
```

### 7.3 Pool Management

```bash
# List available pools
bgpx-cli pools list

# View pool details
bgpx-cli pools show --pool-id bgpx-default

# Add a private pool
bgpx-cli pools add --file my-private-pool.json
```

### 7.4 Domain and Mesh Island Management

```bash
# List routing domains
bgpx-cli domains list

# Find domain bridge nodes
bgpx-cli domains bridges --from clearnet --to mesh:lima-district-1

# View mesh island status
bgpx-cli islands show --island-id lima-district-1
```

### 7.5 Routing Policy Management

```bash
# List current rules
bgpx-cli policy list

# Add a new rule
bgpx-cli policy add --rule '{"destination":"10.20.0.0/16","action":"standard"}' --position 5

# Test which rule matches a destination
bgpx-cli policy test --destination 8.8.8.8

# Reload policy from config file
bgpx-cli policy reload
```

### 7.6 Reputation Monitoring

```bash
# View your node's reputation
bgpx-cli reputation show --self

# View recent reputation events
bgpx-cli reputation events --last 24h
```

---

## 8. SDK Application Development

### 8.1 Connecting to the Local Daemon

For applications using the BGP-X SDK:

**Rust**:
```rust
use bgpx_sdk::Client;

// Auto-discover local daemon
let client = Client::connect_auto().await?;

// Or explicit path
let client = Client::connect("/var/run/bgpx/sdk.sock").await?;
```

**Python**:
```python
import bgpx_sdk

# Auto-discover
client = bgpx_sdk.Client.connect_auto()

# Or explicit
client = bgpx_sdk.Client.connect("/var/run/bgpx/sdk.sock")
```

**TypeScript/JavaScript**:
```typescript
import { Client } from 'bgpx-sdk';

// Auto-discover
const client = await Client.connectAuto();

// Or explicit
const client = await Client.connect('/var/run/bgpx/sdk.sock');
```

### 8.2 Opening a Stream

```rust
// Standard stream to clearnet destination
let stream = client.connect_stream("example.com:443").await?;

// Stream with path constraints
let stream = client.connect_stream_with_path(
    "sensitive.example.com:443",
    PathConfig {
        segments: vec![
            SegmentConfig {
                pool: "bgpx-default".into(),
                hops: 3,
                ..Default::default()
            },
            SegmentConfig {
                pool: "eu-exits".into(),
                hops: 1,
                is_exit: true,
                constraints: SegmentConstraints {
                    require_ech: true,
                    ..Default::default()
                },
            },
        ],
        ..Default::default()
    },
    StreamConfig {
        require_ech: true,
        ..Default::default()
    },
).await?;
```

### 8.3 Cross-Domain Stream to Mesh Service

```rust
// Connect to a .bgpx service hosted in a mesh island
let stream = client.connect_stream_with_path(
    "bgpx://a1b2c3d4...bgpx",  // ServiceID-based address
    PathConfig {
        segments: vec![
            SegmentConfig {
                domain: RoutingDomain::Clearnet,
                hops: 2,
                ..Default::default()
            },
            DomainSegmentConfig {
                segment_type: SegmentType::Bridge,
                from_domain: RoutingDomain::Clearnet,
                to_domain: RoutingDomain::Mesh { 
                    island_id: "lima-district-1".into() 
                },
            },
            SegmentConfig {
                domain: RoutingDomain::Mesh { 
                    island_id: "lima-district-1".into() 
                },
                hops: 2,
                ..Default::default()
            },
        ],
        ..Default::default()
    },
    StreamConfig::default(),
).await?;
```

### 8.4 Registering a BGP-X Native Service

To host a `.bgpx` service from your standalone device:

```rust
// Generate a service keypair
let (service_private_key, service_public_key) = bgpx_sdk::generate_service_keypair();

// Register with the daemon
let service_id = client.register_service(
    "my-service",
    ServiceConfig {
        listen_addr: "127.0.0.1:8080",
        ..Default::default()
    },
).await?;

println!("Service registered: {}", service_id);
// Outputs: ServiceID (hex) which forms the .bgpx address
```

---

## 9. Performance Tuning

### 9.1 Laptop Optimization

Laptops have battery, thermal, and CPU constraints. Optimize for efficiency:

```toml
[sessions]
max_sessions = 500           # Reduced from default 10000
idle_timeout_secs = 60       # Faster cleanup of idle sessions

[paths]
default_hops = 4             # Standard; no need for more on laptop
quality_threshold_rebuild = 0.3  # Tolerate lower quality

[cover_traffic]
enabled = false              # Disabled: saves bandwidth and CPU

[performance]
worker_threads = 2           # Leave other cores for applications

[reputation]
geo_plausibility_enabled = true
geo_plausibility_weight = 0.10  # Reduced weight for battery
```

**Expected performance on modern laptop**:
- Idle CPU: <1%
- Active relay: 5-15% per core
- Memory: 50-150 MB
- Latency overhead: 50-200ms per connection

### 9.2 Desktop Optimization

```toml
[sessions]
max_sessions = 2000

[paths]
default_hops = 4

[cover_traffic]
enabled = true
target_rate_kbps = 10

[performance]
worker_threads = 4           # Use more cores
```

### 9.3 Server Optimization (Relay Node)

```toml
[sessions]
max_sessions = 10000

[paths]
default_hops = 4

[cover_traffic]
enabled = true
target_rate_kbps = 100

[performance]
worker_threads = 8

[tun]
mtu = 1280

[network]
listen_addr = "0.0.0.0"     # Accept from any interface
```

### 9.4 Server Optimization (Exit Node)

See `production/sop.md` for complete exit node configuration. Key settings:

```toml
[node]
role = "exit"

[exit]
enabled = true
policy_path = "/etc/bgpx/exit_policy.toml"

[sessions]
max_sessions = 50000

[performance]
worker_threads = 16

[gateway]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
ech_capable = true
```

---

## 10. Server Deployment

### 10.1 System Hardening

Before deploying a server as a relay or exit node:

```bash
# Disable swap (prevents key material from being swapped to disk)
sudo swapoff -a
echo "vm.swappiness=0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Restrict access to BGP-X files
sudo chmod 700 /etc/bgpx
sudo chmod 700 /var/lib/bgpx
sudo chmod 700 ~/.bgpx  # For user installs

# Firewall configuration
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp      # SSH management
sudo ufw allow 7474/udp    # BGP-X relay
sudo ufw enable
```

### 10.2 SSH Hardening

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Recommended settings:
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes

# Restart SSH
sudo systemctl restart sshd
```

### 10.3 Systemd Service Configuration

For production servers, tune the systemd service:

```bash
# Edit the service file
sudo systemctl edit bgpx-node

# Add:
[Service]
LimitNOFILE=65536
LimitNPROC=4096
CPUQuota=80%
MemoryMax=2G
Restart=always
RestartSec=5
```

### 10.4 Monitoring and Logging

```bash
# Real-time logs
sudo journalctl -u bgpx-node -f

# Prometheus metrics (if enabled)
curl http://localhost:9090/metrics

# Check path quality
bgpx-cli paths list --verbose

# Monitor reputation
watch -n 5 'bgpx-cli reputation show --self'
```

---

## 11. Operating as a Relay Node

### 11.1 Relay Configuration

```toml
[node]
role = "relay"

[network]
listen_addr = "0.0.0.0"
listen_port = 7474

[sessions]
max_sessions = 10000

[paths]
default_hops = 4
```

### 11.2 Bandwidth Considerations

A relay node forwards traffic for other BGP-X users. Estimate your bandwidth contribution:

| Sessions | Avg Traffic | Bandwidth Required |
|---|---|---|
| 100 | Light users | 1 Mbps |
| 500 | Mixed usage | 10 Mbps |
| 1000 | Heavy users | 50 Mbps |
| 5000+ | Relay backbone | 200+ Mbps |

### 11.3 Legal Considerations

As a relay node operator:
- You forward encrypted traffic you cannot read
- You do not see destinations or sources
- You are similar to a Tor relay operator
- Consult local laws regarding relay operation

---

## 12. Operating as an Exit Node

**Important**: Exit nodes have greater legal exposure than relay nodes. See `/legal/liability.md` and `/production/sop.md` before operating an exit node.

### 12.1 Exit Policy

Create `/etc/bgpx/exit_policy.toml`:

```toml
version = 1
logging_policy = "none"  # No logs; critical for user privacy
jurisdiction = "EU-DE"   # Your legal jurisdiction

[[allowed_ports]]
port = 80
protocol = "tcp"

[[allowed_ports]]
port = 443
protocol = "tcp"

[[allowed_ports]]
port = 443
protocol = "udp"  # QUIC/HTTP3

[[denied_destinations]]
cidr = "0.0.0.0/8"
reason = "private_network"

[[denied_destinations]]
cidr = "10.0.0.0/8"
reason = "private_network"

[[denied_destinations]]
cidr = "192.168.0.0/16"
reason = "private_network"

# ECH support
ech_capable = true
```

### 12.2 Exit Node Configuration

```toml
[node]
role = "exit"

[exit]
enabled = true
policy_path = "/etc/bgpx/exit_policy.toml"

[gateway]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

### 12.3 DNS Configuration

Exit nodes resolve DNS on behalf of users. Use encrypted DNS:

```toml
[gateway]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"  # Strip ECS to prevent location leakage
```

---

## 13. Operating as a Domain Bridge Node

A domain bridge node connects two routing domains (e.g., clearnet ↔ mesh island).

### 13.1 Requirements

- **Two network interfaces**: 
  - Clearnet (WAN Ethernet, WiFi, or satellite)
  - Mesh transport (WiFi 802.11s or LoRa radio)
- Sufficient RAM for cross-domain path tables (~2.5 MB per 10,000 entries per domain)

### 13.2 Configuration

```toml
[node]
role = "domain_bridge"

[[routing_domains]]
domain_type = "clearnet"
endpoints = [
    { protocol = "udp", address = "0.0.0.0", port = 7474 }
]

[[routing_domains]]
domain_type = "mesh"
island_id = "lima-district-1"
transports = ["wifi_mesh", "lora"]

# Domain bridge capability is auto-detected when 2+ routing_domains configured
```

### 13.3 Bridge Node Advertisement

Your node will automatically publish a DOMAIN_ADVERTISE record to the unified DHT, indicating it can bridge traffic between clearnet and the specified mesh island.

---

## 14. Geographic Plausibility (Optional)

Geographic plausibility scoring helps verify that nodes are in their claimed regions via RTT measurements. **This is OPTIONAL**.

### 14.1 Declaring Jurisdiction

```toml
[node]
jurisdiction = "EU-DE"  # Optional: declare your legal jurisdiction
```

If you declare a jurisdiction:
- BGP-X will measure RTT to peers
- Compare against expected latency for that region
- Generate reputation events if implausible

If you do NOT declare a jurisdiction:
- Geographic plausibility scoring does NOT apply
- No RTT-based verification

### 14.2 Satellite Connection Exemption

If your standalone device uses satellite internet (Starlink, Iridium, etc.), you are **exempt** from geographic plausibility scoring:

```toml
[node]
jurisdiction = "US"           # Optional
latency_class = "satellite-leo"  # Exempts from geo plausibility
```

---

## 15. Satellite WAN Configuration

If your standalone device uses satellite internet for WAN connectivity:

### 15.1 Starlink Integration

```toml
[network]
# Starlink presents as standard Ethernet
# No special BGP-X configuration needed
# Latency class annotation for metrics
latency_class = "satellite-leo"  # 20-40ms RTT
```

### 15.2 Iridium/Inmarsat Integration

```toml
[network]
latency_class = "satellite-geo"  # 600ms+ RTT

[sessions]
keepalive_interval_secs = 180   # Longer keepalive for high latency
```

**Note**: Satellite internet services (Starlink, Iridium, Inmarsat) are clearnet domain in BGP-X — not a separate satellite routing domain. They provide BGP-routed IP connectivity, just with higher latency.

---

## 16. Cross-Domain Paths

### 16.1 Reaching Mesh Island Services

As a clearnet standalone device, you can reach services hosted in mesh islands without any mesh radio hardware:

```bash
# Connect to a .bgpx service in a mesh island
bgpx-browser https://community-news.mesh.lima-district-1.bgpx
```

The path automatically routes:
1. Clearnet hops to a domain bridge node
2. Bridge node transitions to mesh domain
3. Mesh hops within the island
4. Delivery to the service

### 16.2 Configuring Preferred Bridges

```toml
[domain_bridges]
# Prefer specific bridge nodes
preferred_bridges = [
    { from = "clearnet", to = "mesh:lima-district-1", pool = "trusted-bridges" }
]
```

---

## 17. Running as a System Service vs User Service

### 17.1 System Service (Recommended for servers)

```bash
# Enable
sudo systemctl enable bgpx-node
sudo systemctl start bgpx-node

# Status
sudo systemctl status bgpx-node

# Logs
sudo journalctl -u bgpx-node -f

# Stop
sudo systemctl stop bgpx-node
```

### 17.2 User Service (Recommended for laptops)

```bash
# Enable
systemctl --user enable bgpx-node
systemctl --user start bgpx-node

# Status
systemctl --user status bgpx-node

# Logs
journalctl --user -u bgpx-node -f

# Stop
systemctl --user stop bgpx-node
```

---

## 18. Uninstallation

### 18.1 Package Manager Install

```bash
# Ubuntu/Debian
sudo apt remove bgpx-node bgpx-cli
sudo apt autoremove

# Remove configuration (optional)
rm -rf /etc/bgpx /var/lib/bgpx /var/run/bgpx
```

### 18.2 Binary Install

```bash
# Remove binaries
sudo rm /usr/local/bin/bgpx-node
sudo rm /usr/local/bin/bgpx-cli

# Remove data (optional)
rm -rf /etc/bgpx /var/lib/bgpx ~/.bgpx
```

---

## 19. Troubleshooting

### 19.1 Daemon Won't Start

```bash
# Check logs
sudo journalctl -u bgpx-node -n 100

# Common issues:
# - Permission denied: check /etc/bgpx permissions
# - Port 7474 in use: check with sudo ss -ulnp | grep 7474
# - Invalid config: validate with bgpx-cli config validate
```

### 19.2 No DHT Connectivity

```bash
# Check bootstrap nodes are reachable
ping bootstrap-1.bgpx.network

# Check firewall allows UDP 7474
sudo ufw status

# Check NAT traversal (if behind NAT)
bgpx-cli node dht-stats
```

### 19.3 High CPU/Memory Usage

```bash
# Reduce session count
# Edit /etc/bgpx/config.toml
[sessions]
max_sessions = 500

# Reduce worker threads
[performance]
worker_threads = 2
```

### 19.4 BGP-X Browser Can't Connect

```bash
# Check socket exists
ls -la /var/run/bgpx/sdk.sock

# Check daemon is running
bgpx-cli node stats

# Try TCP endpoint
# Edit /etc/bgpx/config.toml
[sdk_api]
tcp_listen = "127.0.0.1:7475"

# Restart daemon
sudo systemctl restart bgpx-node
```

---

## 20. Next Steps

- **Understand routing policy**: `docs/routing_policy.md`
- **Deploy a relay node**: `production/sop.md`
- **Deploy an exit node**: `gateway/exit_node.md`
- **Connect to mesh islands**: `docs/mesh_architecture.md`
- **BGP-X Browser guide**: `application/bgpx_browser_user_guide.md`
- **SDK development**: `sdk/sdk_spec.md`
