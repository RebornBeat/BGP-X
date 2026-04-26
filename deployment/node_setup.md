# BGP-X Node Setup Guide

**Version**: 0.1.0-draft

---

## 1. Prerequisites

| Platform | Requirements |
|---|---|
| Linux (x86_64) | Ubuntu 22.04+, Debian 12+, or equivalent |
| Linux (ARM64) | Ubuntu 22.04+ arm64 (Raspberry Pi, GL.iNet) |
| OpenWrt | OpenWrt 23.05+ on MT7981B or newer |
| Embedded (mesh-only) | bgpx-mesh-firmware package |

Memory: ≥ 256 MB for relay; ≥ 512 MB for domain bridge; ≥ 64 MB for mesh-only firmware.

---

## 2. Installation

```bash
# Ubuntu/Debian
curl -fsSL https://pkg.bgpx.network/gpg.pub | sudo gpg --dearmor -o /usr/share/keyrings/bgpx.gpg
echo "deb [signed-by=/usr/share/keyrings/bgpx.gpg] https://pkg.bgpx.network/apt stable main" \
    | sudo tee /etc/apt/sources.list.d/bgpx.list
sudo apt update && sudo apt install bgpx-node

# Build from source
cargo build --release --bin bgpx-node
sudo install -m 755 target/release/bgpx-node /usr/local/bin/bgpx-node
```

---

## 3. Initial Configuration

```bash
# Generate node identity
bgpx-node keygen --output /etc/bgpx/node_private_key
chmod 600 /etc/bgpx/node_private_key

# Create configuration
cat > /etc/bgpx/node.toml << 'EOF'
[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
name = "my-bgpx-node"
data_dir = "/var/lib/bgpx"

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
public_addr = "YOUR_PUBLIC_IP_HERE"
public_port = 7474

[dht]
bootstrap_nodes = []  # Uses built-in defaults
EOF

# Validate configuration
bgpx-node --config /etc/bgpx/node.toml --config-check
```

---

## 4. Firewall Configuration

```bash
# Allow BGP-X traffic
ufw allow 7474/udp comment "BGP-X overlay"

# For bridge nodes: also allow mesh traffic
# WiFi mesh: handled at L2, no firewall rules needed
# LoRa: serial device access (no network firewall rules)
```

---

## 5. systemd Service

```bash
sudo cp /usr/share/bgpx/bgpx-node.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable bgpx-node
sudo systemctl start bgpx-node
sudo systemctl status bgpx-node
```

---

## 6. Exit Node Additional Setup

```bash
# Configure exit policy
cat >> /etc/bgpx/node.toml << 'EOF'

[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = [
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16", "127.0.0.0/8"
]
logging_policy = "none"
operator_contact = "ops@example.com"
jurisdiction = "DE"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
EOF

# Update roles
sed -i 's/roles = \["relay", "entry", "discovery"\]/roles = ["relay", "entry", "exit", "discovery"]/' \
    /etc/bgpx/node.toml
```

---

## 7. Domain Bridge Node Setup

If your node has both internet and mesh radio connectivity:

```bash
cat >> /etc/bgpx/node.toml << 'EOF'

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-name"
enabled = true
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"
# lora_interface = "/dev/ttyUSB0"
# lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true

[mesh_island]
auto_publish_advertisement = true
advertisement_interval_hours = 8
island_dht_sync = true
EOF

# Verify domain configuration
bgpx-cli domains list
bgpx-cli domains island-status --island-id your-island-name
bgpx-cli domains bridges --from clearnet --to "mesh:your-island-name"
```

---

## 8. Verification

```bash
# Check node status
bgpx-cli status

# Verify node advertisement published
bgpx-cli nodes get --self

# Check DHT participation
bgpx-cli dht status

# Build a test path
bgpx-cli paths build --segments "default:3,default:1"

# For bridge nodes: verify bridge pair
bgpx-cli domains bridges --from clearnet --to "mesh:your-island-name"
bgpx-cli domains test --from clearnet --to "mesh:your-island-name"
```

---

## 9. Monitoring

```bash
# Enable Prometheus metrics
cat >> /etc/bgpx/node.toml << 'EOF'

[metrics]
prometheus_enabled = true
prometheus_addr = "127.0.0.1"
prometheus_port = 9474
EOF

# Query metrics
curl -s http://127.0.0.1:9474/metrics | grep bgpx_
```

---

## 10. OpenWrt Specific

```bash
# Install on OpenWrt
opkg update && opkg install bgpx-node

# Configuration at /etc/bgpx/node.toml
# Data directory at /var/lib/bgpx (tmpfs — use /overlay for persistence)

# Enable service
/etc/init.d/bgpx-node enable
/etc/init.d/bgpx-node start
```
