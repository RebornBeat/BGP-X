# BGP-X Node Daemon Specification

**Version**: 0.1.0-draft

---

## 1. Deployment Context

The BGP-X node daemon (`bgpx-node`) is the single BGP-X routing stack and inter-protocol domain router. It runs on a network router or standalone device. All other BGP-X components — SDK applications, the configuration client — are clients of this daemon.

The same daemon binary runs on all Tier 1 hardware (BGP-X Router v1, BGP-X Node v1, BGP-X Gateway v1) and compatible third-party hardware. The differences between products are form factor, radio configuration, and deployment context — not daemon capability.

### 1.1 Hardware Deployment Modes

**BGP-X Router v1** (All-In-One):
- **Target**: End users, families, home offices, small organizations.
- **Role**: Replaces a standard home or office router entirely.
- **Daemon Config**: Operates in dual-stack mode (BGP + BGP-X) or BGP-X-only mode.
- **Key Features**: WAN + LAN switching + integrated LoRa/WiFi mesh/BLE + domain bridge capability.
- **User Experience**: Connects ISP WAN and all LAN devices. No per-device configuration needed.

**BGP-X Node v1** (Community Contributor):
- **Target**: Community mesh contributors, outdoor relay operators, range extenders.
- **Role**: Runs full `bgpx-node` daemon. **Not a "lite" or partial device.**
- **Daemon Config**: Operates in Mesh Relay, Domain Bridge, Range Extension, or Community Gateway mode.
- **Key Features**: Solar/battery optimized, outdoor IP67 enclosure, configurable radio mix.
- **User Experience**: Added to existing network to expand mesh coverage. Does not replace home router.

**BGP-X Gateway v1** (Provider Infrastructure):
- **Target**: ISPs, hosting providers, enterprise operators.
- **Role**: High-throughput provider-grade exit/relay infrastructure.
- **Daemon Config**: Optimized for >1 Gbps clearnet throughput, high concurrent session capacity.
- **Key Features**: Rack-mountable, SFP+ uplink, 2×2.5GbE WAN, no integrated LoRa (USB available).
- **User Experience**: Colocated in datacenters or operated as remote infrastructure.

**BGP-X Client Node** (Tier 2, Reference Hardware):
- **Target**: Individual users, IoT sensors, portable field operation.
- **Role**: Endpoint only. Does NOT run `bgpx-node` daemon.
- **Firmware**: `bgpx-client-firmware` (subset implementation for ESP32-S3 / nRF52840).
- **Key Features**: Low-cost ($25-40), battery-native, LoRa + WiFi + BLE.
- **User Experience**: Connects to mesh island as client. No routing for others.

**BGP-X Adapter/Dongle** (Tier 3):
- **Target**: Users with existing computers wanting mesh connectivity.
- **Role**: USB LoRa modem. No routing intelligence.
- **Firmware**: `bgpx-modem-firmware`.
- **Key Features**: USB-C dongle ($15-25), transparent pipe to LoRa radio.
- **User Experience**: Plug into laptop/desktop; host runs `bgpx-node`.

**OpenWrt Package (Software-Only)**:
- **Target**: Users with compatible third-party routers (GL.iNet, Raspberry Pi, etc.).
- **Role**: Software package installing `bgpx-node` on existing hardware.
- **Key Features**: Domain bridge possible with USB adapters.

### 1.2 Deployment Roles (Daemon Configuration Modes)

A single `bgpx-node` daemon can simultaneously serve multiple roles based on configuration:

- **Relay**: Forwards onion-encrypted packets within a single routing domain.
- **Entry Node**: Accepts client connections; knows client transport address.
- **Exit Node / Gateway**: Terminates overlay path, connects to clearnet, enforces exit policy.
- **Discovery Node**: Participates in DHT, stores records.
- **Domain Bridge Node**: Has endpoints in 2+ routing domains; forwards traffic between them.
- **Mesh Node**: Operates without ISP via mesh transport (WiFi mesh, LoRa, BLE).
- **Mesh Island Gateway**: Domain bridge node specifically connecting a mesh island to clearnet.
- **Range Extension Node**: Prioritizes forwarding over session origination (software configuration of Node v1).

A node automatically becomes a **domain bridge node** when configured with two or more `[[routing_domains]]` entries.

---

## 2. Architecture

The daemon is structured as a set of cooperating async tasks within a single process. The reference implementation uses Rust with Tokio as the async runtime.

```
bgpx-node process
├── Main task (signal handler, lifecycle coordinator)
│
├── Network I/O subsystem
│   ├── UDP listener (clearnet transport, SO_REUSEPORT)
│   │   ├── Receiver threads (N = CPU cores / 2)
│   │   └── Sender thread pool (4 threads)
│   ├── Mesh transport handlers (per domain/island)
│   │   ├── WiFi mesh interface (802.11s)
│   │   ├── LoRa serial interface
│   │   └── BLE GATT interface
│   └── Satellite transport handler (if USB satellite modem detected)
│
├── Forwarding pipeline
│   ├── Receiver threads
│   ├── Session table (sharded concurrent hash map, 64 shards)
│   ├── Single-domain path table (path_id → predecessor)
│   ├── Cross-domain path table (path_id → {src_domain, src_addr, dst_domain})
│   └── Worker threads (dispatch, output queue)
│
├── Domain Manager (NEW)
│   ├── Routing domain registry (which domains this node serves)
│   ├── Domain bridge manager
│   │   ├── Cross-domain path_id table
│   │   ├── Domain transition handler (DOMAIN_BRIDGE hop processing)
│   │   └── Bridge availability tracker (per bridge pair)
│   ├── Mesh island manager
│   │   ├── Island registry
│   │   ├── Island DHT cache
│   │   └── Island advertisement publisher
│   └── Transport selector (correct transport per domain)
│
├── Session manager
│   ├── Handshake handler (domain-agnostic)
│   ├── Keepalive monitor
│   └── Session re-handshake scheduler (24hr)
│
├── DHT subsystem (Unified, domain-aware queries)
│   ├── Routing table (k-bucket structure)
│   ├── Advertisement publisher
│   ├── Pool manager
│   └── Bootstrap manager
│
├── Routing Policy Engine (dual-stack mode)
│   ├── Rule evaluator
│   └── DNS routing policy
│
├── SDK Socket server (bgpx-sdk clients)
│
├── Control API server (bgpx-cli and management tools)
│
├── Reputation system
│   ├── Local reputation database
│   └── Geographic plausibility scorer (OPTIONAL)
│
├── Exit policy engine (exit/gateway nodes only)
│   ├── Policy evaluator
│   ├── DNS resolver (DoH with DNSSEC + ECS stripping)
│   ├── ECH manager
│   └── Outbound connection manager
│
├── Pluggable transport (when enabled)
│   └── PT subprocess manager
│
└── Metrics collector
```

---

## 3. Configuration

Default location: `/etc/bgpx/node.toml` (system) or `~/.config/bgpx/node.toml` (user)

### Complete Configuration Reference

```toml
# BGP-X Node Configuration
# All values shown are defaults unless marked REQUIRED

[node]
# REQUIRED: Path to the node's Ed25519 private key file
private_key_path = "/etc/bgpx/node_private_key"

# REQUIRED: Roles this node serves
roles = ["relay", "entry", "discovery"]

# Human-readable node name (used in logs only, not published)
name = "bgpx-node-1"

# Data directory for persistent state
data_dir = "/var/lib/bgpx"

# Log level: "error", "warn", "info" (no debug/trace in production)
log_level = "info"

# Log format: "text" or "json"
log_format = "json"

# Log destination: "stdout", "syslog", or file path
log_destination = "stdout"

# ─────────────────────────────────────────────────────────────────────────────
# ROUTING DOMAINS
# A node may serve one or more routing domains. Nodes with 2+ entries
# automatically become domain bridge nodes.
# ─────────────────────────────────────────────────────────────────────────────

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
listen_addr_v6 = "::"
public_addr = ""          # Set if behind NAT
public_port = 7474

# Mesh island (add additional [[routing_domains]] for bridge nodes)
# [[routing_domains]]
# domain_type = "mesh"
# island_id = "lima-district-1"
# enabled = true
# transports = ["wifi_mesh", "lora"]
# wifi_mesh_interface = "mesh0"
# lora_interface = "/dev/ttyUSB0"
# lora_frequency_mhz = 868.0
# lora_spreading_factor = 7

# ─────────────────────────────────────────────────────────────────────────────
# DOMAIN BRIDGE CONFIGURATION
# Automatically detected if 2+ routing_domains configured
# ─────────────────────────────────────────────────────────────────────────────

[domain_bridge]
enabled = false
# bridges = [
#     { from = "clearnet", to = "mesh:lima-district-1" }
# ]

# ─────────────────────────────────────────────────────────────────────────────
# MESH ISLAND CONFIGURATION (for gateway/bridge nodes)
# ─────────────────────────────────────────────────────────────────────────────

[mesh_island]
auto_publish_advertisement = true
advertisement_interval_hours = 8
island_dht_sync = true

# ─────────────────────────────────────────────────────────────────────────────
# NETWORK CONFIGURATION
# ─────────────────────────────────────────────────────────────────────────────

[network]
socket_recv_buffer = 16777216  # 16 MB
socket_send_buffer = 16777216

# ─────────────────────────────────────────────────────────────────────────────
# SESSION MANAGEMENT
# ─────────────────────────────────────────────────────────────────────────────

[sessions]
max_sessions = 10000
session_idle_timeout_seconds = 90
session_idle_timeout_lora_seconds = 300  # Extended for high-latency mesh
keepalive_interval_seconds = 25          # Randomized ±5s by daemon (MANDATORY)
handshake_timeout_seconds = 10
handshake_timeout_lora_seconds = 60      # Extended for mesh
max_session_bandwidth_mbps = 0           # 0 = unlimited
session_rehandshake_age_hours = 24       # Trigger re-handshake for forward secrecy

# ─────────────────────────────────────────────────────────────────────────────
# BANDWIDTH MANAGEMENT
# ─────────────────────────────────────────────────────────────────────────────

[bandwidth]
total_bandwidth_mbps = 0                 # 0 = unlimited
reserved_bandwidth_mbps = 10             # Reserved for DHT/control

# ─────────────────────────────────────────────────────────────────────────────
# DHT CONFIGURATION
# ─────────────────────────────────────────────────────────────────────────────

[dht]
bootstrap_nodes = [
    "nodeid@ip:port",
]
k = 20
alpha = 3
refresh_interval_minutes = 60
republish_interval_hours = 12
advertisement_validity_hours = 24
max_stored_records = 10000

# ─────────────────────────────────────────────────────────────────────────────
# NODE ADVERTISEMENT
# ─────────────────────────────────────────────────────────────────────────────

[advertisement]
bandwidth_mbps = 100
latency_ms = 12
region = "EU"
country = "DE"             # OPTIONAL: jurisdiction declaration for geo plausibility
asn = 12345
operator_id = ""
operator_private_key_path = ""

# ─────────────────────────────────────────────────────────────────────────────
# EXIT POLICY (exit/gateway nodes only)
# ─────────────────────────────────────────────────────────────────────────────

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
    "::1/128",
    "fc00::/7",
    "fe80::/10"
]
logging_policy = "none"
operator_contact = ""
jurisdiction = "DE"
max_outbound_connections = 2000
outbound_connect_timeout_seconds = 10
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"    # Mandatory for exit nodes
ech_capable = true

# ─────────────────────────────────────────────────────────────────────────────
# DHT POOLS
# ─────────────────────────────────────────────────────────────────────────────

[pools]
member_pools = []
discover_pools = ["bgpx-default"]
pool_minimum_members = 5
fallback_to_default = true

# ─────────────────────────────────────────────────────────────────────────────
# SDK API (for native applications)
# ─────────────────────────────────────────────────────────────────────────────

[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"
tcp_listen = ""            # e.g., "192.168.1.1:7475" for LAN access
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/sdk_auth_token"

# ─────────────────────────────────────────────────────────────────────────────
# ROUTING POLICY (dual-stack mode)
# ─────────────────────────────────────────────────────────────────────────────

[routing_policy]
policy_file = "/etc/bgpx/routing_policy.toml"
default_action = "bgpx"

# ─────────────────────────────────────────────────────────────────────────────
# CONTROL API
# ─────────────────────────────────────────────────────────────────────────────

[control_api]
socket_path = "/var/run/bgpx/control.sock"
socket_permissions = "0600"

# ─────────────────────────────────────────────────────────────────────────────
# REPUTATION SYSTEM
# ─────────────────────────────────────────────────────────────────────────────

[reputation]
geo_plausibility_enabled = true
geo_plausibility_warn_threshold = 2.0
geo_plausibility_alert_threshold = 3.0
enforce_global_blacklist = false
blacklist_default_duration_days = 30
blacklist_auto_review = true

# ─────────────────────────────────────────────────────────────────────────────
# PLUGGABLE TRANSPORT
# ─────────────────────────────────────────────────────────────────────────────

[pluggable_transport]
enabled = false
mode = "builtin"          # "builtin" or "external"
binary_path = ""
listen_addr = "127.0.0.1"
listen_port = 7475
timing_jitter_max_ms = 50
max_padding_bytes = 200
apply_to_mesh = false

# ─────────────────────────────────────────────────────────────────────────────
# MESH TRANSPORT
# ─────────────────────────────────────────────────────────────────────────────

[mesh]
enabled = false
transports = []            # ["wifi_mesh", "lora", "ble"]
wifi_mesh_interface = ""
lora_interface = ""
lora_frequency_mhz = 868.0
beacon_interval_seconds = 30

# ─────────────────────────────────────────────────────────────────────────────
# EXTENSIONS
# ─────────────────────────────────────────────────────────────────────────────

[extensions]
cover_traffic_enabled = false
cover_traffic_target_kbps = 100
path_quality_reporting_enabled = true
cross_domain_routing_enabled = true
domain_bridge_enabled = true

# ─────────────────────────────────────────────────────────────────────────────
# STORE-AND-FORWARD (for LoRa/satellite)
# ─────────────────────────────────────────────────────────────────────────────

[store_and_forward]
enabled = false
buffer_size_kb = 1024
packet_ttl_seconds = 3600

# ─────────────────────────────────────────────────────────────────────────────
# METRICS
# ─────────────────────────────────────────────────────────────────────────────

[metrics]
prometheus_enabled = false
prometheus_addr = "127.0.0.1"
prometheus_port = 9474
```

---

## 4. Logging Policy

### Valid Production Log Levels

| Level | Description |
|---|---|
| `error` | Critical failures only |
| `warn` | Operational warnings (exit node default) |
| `info` | Normal operational events |

`debug` and `trace` are **NOT valid in production builds**. They expose internal state that could leak information.

### Prohibited Log Content

The following **MUST NEVER** appear in any log file under any log level:

- Client IP addresses at relay or exit nodes
- Destination addresses (full) at any node
- `path_id` values
- Path composition (which nodes appeared in same path)
- Session identifiers
- Traffic volume per path or per session
- Pool query history
- Application data content
- Specific denied destination addresses (log denial category only)
- Which routing domains a specific session traversed
- Cross-domain path composition
- Mesh island identifiers associated with specific sessions

---

## 5. Startup Sequence

```
1.  Parse and validate configuration
    ├── Verify required fields
    ├── Verify private key file readable and valid
    └── If exit role: verify exit policy complete

2.  Load node identity
    ├── Load Ed25519 private key from private_key_path
    ├── Derive public key and NodeID
    └── Log NodeID at INFO level (NodeID is public information)

3.  Initialize data directory
    ├── Create data_dir if not exists
    ├── Load reputation database
    └── Load DHT routing table

4.  Initialize network subsystem
    ├── Bind UDP socket
    ├── Bind IPv6 socket if configured
    └── If mesh enabled: initialize mesh transport(s)

5.  Initialize session manager
    └── Start keepalive monitor background task

6.  Initialize pool manager
    ├── Load configured member pools
    ├── Verify pool signatures
    └── Update node advertisement with pool memberships

7.  Initialize domain manager
    ├── For each [[routing_domains]] entry: initialize appropriate transport
    ├── If bridge_capable (2+ domains): initialize domain bridge manager
    └── If mesh domains: initialize mesh island manager

8.  Initialize DHT subsystem
    ├── Load routing table from disk
    ├── If internet mode: connect to bootstrap nodes
    ├── If mesh mode: broadcast MESH_BEACON and collect responses
    └── Publish node advertisement

9.  If bridge_capable: publish DOMAIN_ADVERTISE for each bridge pair

10. If island manager and internet available: publish MESH_ISLAND_ADVERTISE

11. Initialize exit policy engine (if exit/gateway role)

12. Initialize routing policy engine (if dual-stack mode)

13. Initialize pluggable transport (if enabled)

14. Initialize SDK socket server

15. Initialize Control API server

16. Initialize metrics exporter (if enabled)

17. Initialize session re-handshake scheduler (24hr background task)

18. Enter ACTIVE state
    └── Log "Node active. NodeID: <node_id>"
```

---

## 6. Graceful Shutdown (SIGTERM/SIGINT)

```
1. Transition to DRAIN state
   a. Publish NODE_WITHDRAW
   b. If bridge node: set bridge.available = false in DOMAIN_ADVERTISE and re-publish
   c. Wait up to 30 seconds for DHT propagation

2. Stop accepting new sessions (reject HANDSHAKE_INIT)

3. Wait for in-flight sessions to complete
   ├── Send KEEPALIVE to all active sessions
   ├── Wait up to shutdown_timeout_seconds (default 30)
   └── After timeout: force-close remaining sessions

4. Flush reputation database to disk

5. Flush DHT routing table to disk

6. Remove Control API and SDK socket files

7. Zeroize all session keys in memory (MANDATORY)
   ├── session_key for all active sessions
   └── keepalive_key for all active sessions

8. Exit with code 0
```

---

## 7. SIGHUP (Config Reload)

On receiving SIGHUP, daemon reloads configuration file. Live-reloadable options:

- `max_sessions`, `total_bandwidth_mbps`, `max_session_bandwidth_mbps`
- `log_level`, `cover_traffic_enabled`, `cover_traffic_target_kbps`
- `exit_policy.allow_ports` (new streams), `exit_policy.deny_destinations` (new streams)
- Pool discovery configuration
- Routing policy rules
- Domain bridge `bridge.available` states
- `mesh_island.advertisement_interval_hours`

**Requires restart** (not live-reloadable):
- `private_key_path`, `listen_addr`, `listen_port`, `data_dir`
- `control_api.socket_path`, `sdk_api.socket_path`
- `[[routing_domains]]` entries

---

## 8. Key Management

### Key Generation

```bash
bgpx-cli keygen --output /etc/bgpx/node_private_key
```

Output: 32-byte Ed25519 seed, permissions 0600.

### Key Storage Requirements

- Readable only by bgpx-node process user (permissions 0600)
- On stable filesystem (not `/tmp` or network filesystem)
- **MUST** be backed up — loss means loss of node identity and reputation

For production: store in HSM or secrets manager (HashiCorp Vault, etc.).

### Key Rotation

Rotating node private key changes NodeID — equivalent to new node identity. Reputation is **NOT** transferable.

Procedure:
1. Generate new keypair
2. Update `private_key_path` configuration
3. Restart daemon
4. Old NodeID's reputation is abandoned

---

## 9. Resource Requirements

| Resource | Relay Only | Domain Bridge (2 domains) | Gateway |
|---|---|---|---|
| RAM | 256 MB minimum | 512 MB minimum | 1 GB minimum |
| CPU | 1 core | 2 cores | 4 cores |
| Disk | 1 GB | 2 GB | 5 GB |
| Network | 10 Mbps symmetric | 100 Mbps symmetric | 1 Gbps symmetric |

### Memory Scaling

```
estimated_MB ≈ 50 + (session_count × 0.256) + (cross_domain_path_entries × 0.1)
```

At 10,000 sessions: approximately 2,610 MB.

---

## 10. Security Hardening

### systemd Service

```ini
[Service]
User=bgpx
Group=bgpx
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/bgpx /var/run/bgpx /var/log/bgpx
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
SystemCallFilter=@system-service
MemoryDenyWriteExecute=true
RestrictNamespaces=true
RestrictRealtime=true
LockPersonality=true
ProtectKernelTunables=true
ProtectControlGroups=true
LimitNOFILE=65536
```

### Firewall Rules (Relay Node)

```bash
# Allow BGP-X overlay traffic
ufw allow 7474/udp

# Allow SSH
ufw allow 22/tcp

# Enable
ufw enable
```

### Kernel Parameters

```ini
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.netdev_max_backlog = 65536
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
net.ipv4.conf.all.rp_filter = 1
```

---

## 11. Monitoring

### Prometheus Metrics

```
# Core metrics
bgpx_sessions_active                    # Gauge
bgpx_packets_relayed_total              # Counter
bgpx_bytes_relayed_total                # Counter
bgpx_handshake_total{result="..."}      # Counter by result
bgpx_dht_routing_table_size             # Gauge
bgpx_reputation_blacklisted_nodes       # Gauge

# Path metrics
bgpx_path_construction_total{result}    # Counter
bgpx_return_traffic_forwarded_total     # Counter
bgpx_session_rehandshakes_total         # Counter

# Geo plausibility
bgpx_geo_plausibility_alerts_total      # Counter

# Pools
bgpx_pool_members_active{pool_id}       # Gauge per pool

# Exit/Gateway
bgpx_ech_connections_total              # Counter
bgpx_exit_connections_active            # Gauge

# Cross-domain
bgpx_routing_domains_active             # Count of active routing domains
bgpx_domain_bridge_transitions_total    # Total domain bridge transitions
bgpx_cross_domain_paths_active         # Active cross-domain paths
bgpx_mesh_island_reachable{island}      # Boolean per island
bgpx_bridge_pairs_active               # Count of active bridge pairs
bgpx_cross_domain_path_table_size      # Entries in cross-domain path table
bgpx_bridge_availability{from,to}      # Bridge pair availability (0/1)

# Mesh
bgpx_mesh_peers_active                  # Gauge
bgpx_pt_active                          # Boolean
```

---

## 12. Deployment Checklist

Before going live:

- [ ] Node private key generated and securely backed up
- [ ] Configuration complete and validated (`bgpx-node --config-check`)
- [ ] Firewall rules applied
- [ ] systemd service installed with hardening
- [ ] Public IP and port reachable from internet
- [ ] If exit node: exit policy reviewed, operator contact set, DoH configured, ECH tested
- [ ] If bridge node: both `routing_domains` configured, bridge pair validated
- [ ] If mesh island gateway: `island_id` chosen (unique), `MESH_ISLAND_ADVERTISE` publishing verified
- [ ] Pool memberships configured (if applicable)
- [ ] Prometheus monitoring configured
- [ ] Log rotation configured
- [ ] Backup procedure tested for `/var/lib/bgpx/`
- [ ] Pool curator key backed up separately (if pool curator)
- [ ] Hardware matches targeted deployment mode (Router v1, Node v1, Gateway v1, etc.)
