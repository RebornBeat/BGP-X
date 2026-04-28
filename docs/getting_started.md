# BGP-X Getting Started Guide

**Version**: 0.1.0-draft

---

## 1. What You're Setting Up

BGP-X runs on your router or device and routes traffic through an onion-encrypted overlay network. No app modifications needed. All devices on your network are protected.

---

## 2. Choose Your Deployment Mode

| Mode | What You Need | What You Get |
|---|---|---|
| **Router (Mode 1)** | OpenWrt router | All LAN devices protected |
| **Standalone (Mode 3)** | Any Linux device | That device protected |
| **Mesh Node (Mode 4)** | BGP-X hardware or OpenWrt + radio | ISP-free networking |
| **Gateway (Mode 5)** | Router with internet + mesh radio | Bridge mesh island to internet |

---

## 3. Quick Start (Linux Standalone)

```bash
# Install
curl -fsSL https://install.bgpx.network/quick.sh | bash

# Generate identity and default config
bgpx-node setup --output /etc/bgpx/node.toml

# Edit config: set your public IP
nano /etc/bgpx/node.toml

# Start
sudo systemctl enable --now bgpx-node

# Verify
bgpx-cli status
bgpx-cli paths build --segments "default:3,default:1"
```

---

## 4. Quick Start (OpenWrt Router)

```bash
# On the router
opkg update && opkg install bgpx-node luci-app-bgpx

# Configure via LuCI web interface
# Or via CLI:
uci set bgpx.@node[0].enabled=1
uci set bgpx.@node[0].public_addr=$(curl -s ifconfig.me)
uci commit bgpx
/etc/init.d/bgpx-node restart
```

---

## 5. Three Equal Entry Points

BGP-X treats three network classes as equal first-class citizens:

**Clearnet (standard internet)**: you have a standard ISP connection. BGP-X daemon routes your traffic through the onion overlay. No special hardware needed.

**BGP-X overlay**: the onion-encrypted routing layer itself. Paths stay within the overlay for additional hops before exiting.

**Mesh islands**: community radio networks (WiFi, LoRa, Bluetooth). No ISP required within the island. Cross-domain access to/from clearnet via bridge nodes.

---

## 6. N-Hop Unlimited

BGP-X imposes no protocol maximum on path length. Default is 4 hops for single-domain clearnet. You can specify paths with any number of hops across any combination of routing domains.

```toml
# High-security path: 15 hops across 3 domains
domain_segments = [
    { type = "segment", domain = "clearnet", hops = 5 },
    { type = "bridge", from_domain = "clearnet", to_domain = "mesh:trusted-island" },
    { type = "segment", domain = "mesh:trusted-island", hops = 4 },
    { type = "bridge", from_domain = "mesh:trusted-island", to_domain = "clearnet" },
    { type = "segment", domain = "clearnet", pool = "private-exits", hops = 4, exit = true }
]
```

---

## 7. Cross-Domain Routing

BGP-X enables routing between clearnet, overlay, and mesh islands in any combination.

**Clearnet user reaching mesh island service**:
- No mesh hardware needed
- BGP-X daemon discovers bridge nodes automatically
- Bridge node handles radio transmission on your behalf

**Mesh island reaching clearnet**:
- Route via gateway node (domain bridge with internet connection)
- No direct ISP exposure for island members

**See `/docs/cross_domain_routing.md` for detailed examples.**

---

## 8. Basic Routing Policy

```toml
# /etc/bgpx/routing_policy.toml

[[rules]]
id = "default"
match = { destination_domain = ["*"] }
action = "bgpx"
path_constraints = {
    exit_logging_policy = "none",
    exit_jurisdiction_blacklist = ["US"]
}
default = true

[[rules]]
id = "direct-local"
match = { destination_cidr = ["192.168.0.0/16", "10.0.0.0/8"] }
action = "standard"
priority = 0
```

---

## 9. First Verification

```bash
# Check node is active and DHT-connected
bgpx-cli status

# Check you have nodes in database
bgpx-cli nodes list --limit 5

# Build a test path and verify it works
bgpx-cli paths build --segments "default:3,default:1"
bgpx-cli paths list

# Test a connection
curl --proxy socks5h://127.0.0.1:1080 https://check.bgpx.network/

# If domain bridge configured:
bgpx-cli domains list
bgpx-cli domains test --from clearnet --to "mesh:your-island-name"
```

---

## 10. Next Steps

- **Exit node**: see `/deployment/node_setup.md` Section 6
- **Domain bridge**: see `/deployment/node_setup.md` Section 7
- **Mesh island**: see `/deployment/mesh_deployment.md`
- **SDK integration**: see `/sdk/sdk_spec.md`
- **Routing policy**: see `/docs/routing_policy.md`
- **Security**: see `/security/threat_model.md`
