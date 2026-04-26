# BGP-X Node Daemon Specification

**Version**: 0.1.0-draft

---

## 1. Deployment Context

The BGP-X node daemon (`bgpx-node`) is the single BGP-X routing stack and inter-protocol domain router. It runs on a network router or standalone device. All other components are clients of this daemon.

### Deployment Roles

All six existing deployment modes are retained. Additionally, any node with multiple `[[routing_domains]]` entries configured is automatically a **domain bridge node**.

A single bgpx-node daemon can simultaneously be:
- A clearnet relay
- A mesh island gateway (bridge to clearnet)
- A mesh-to-mesh bridge (between two mesh islands)

---

## 2. Architecture

```
bgpx-node process
├── Main task (signal handler, lifecycle coordinator)
│
├── Network I/O subsystem
│   ├── UDP listener (clearnet transport, SO_REUSEPORT)
│   ├── Mesh transport handlers (per domain/island)
│   ├── Satellite transport handler (if configured)
│   └── Sender thread pool (4 threads, domain-aware)
│
├── Domain Manager (NEW)
│   ├── Routing domain registry (which domains this node serves)
│   ├── Domain bridge manager
│   │   ├── Cross-domain path_id table (path_id → domain pair routing)
│   │   ├── Domain bridge transition handler (DOMAIN_BRIDGE hop processing)
│   │   └── Bridge availability tracker (monitors each bridge pair)
│   ├── Mesh island manager
│   │   ├── Island registry (known mesh islands)
│   │   ├── Island DHT cache
│   │   └── Island advertisement publisher
│   └── Transport selector (correct transport per domain)
│
├── Forwarding pipeline (DOMAIN_BRIDGE hop type support added)
│
├── Session manager (domain-agnostic)
│
├── DHT subsystem (unified, domain-aware queries)
│
├── Routing Policy Engine (domain-aware rules)
│
├── SDK Socket server
│
├── Control API server (domain management methods added)
│
├── Reputation system (domain-aware event tagging)
│
├── Exit policy engine (exit/gateway nodes only)
│
├── Pluggable transport (when enabled)
│
└── Metrics collector
```

---

## 3. Configuration

```toml
# BGP-X Node Configuration

[node]
private_key_path = "/etc/bgpx/node_private_key"
roles = ["relay", "entry", "discovery"]
name = "my-bgpx-relay-1"
data_dir = "/var/lib/bgpx"
log_level = "info"
log_format = "json"
log_destination = "/var/log/bgpx/node.log"

# Routing domains this node serves
# For single-domain clearnet node: one [[routing_domains]] entry
# For bridge node: two or more [[routing_domains]] entries

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
listen_addr_v6 = "::"
public_addr = ""
public_port = 7474

# Mesh island (add for bridge/gateway nodes):
# [[routing_domains]]
# domain_type = "mesh"
# island_id = "my-island-name"
# enabled = true
# transports = ["wifi_mesh", "lora"]
# wifi_mesh_interface = "mesh0"
# lora_interface = "/dev/ttyUSB0"
# lora_frequency_mhz = 868.0
# lora_spreading_factor = 7

# Domain bridge (auto-detected from routing_domains, or explicit):
[domain_bridge]
enabled = false
# bridges = [
#     { from = "clearnet", to = "mesh:my-island-name" }
# ]

[mesh_island]
auto_publish_advertisement = true
advertisement_interval_hours = 8
island_dht_sync = true

[sessions]
max_sessions = 10000
session_idle_timeout_seconds = 90
session_idle_timeout_lora_seconds = 300
keepalive_interval_seconds = 25
handshake_timeout_seconds = 10
handshake_timeout_lora_seconds = 60
max_session_bandwidth_mbps = 0
session_rehandshake_age_hours = 24

[bandwidth]
total_bandwidth_mbps = 0
reserved_bandwidth_mbps = 10

[dht]
bootstrap_nodes = []
k = 20
alpha = 3
refresh_interval_minutes = 60
republish_interval_hours = 12
advertisement_validity_hours = 24
max_stored_records = 10000

[advertisement]
bandwidth_mbps = 100
latency_ms = 12
region = "EU"
country = "DE"
asn = 12345
operator_id = ""
operator_private_key_path = ""

[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = [
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "127.0.0.0/8", "169.254.0.0/16", "100.64.0.0/10",
    "::1/128", "fc00::/7", "fe80::/10"
]
logging_policy = "none"
operator_contact = ""
jurisdiction = "DE"
max_outbound_connections = 2000
outbound_connect_timeout_seconds = 10
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true

[pools]
member_pools = []
discover_pools = ["bgpx-default"]
pool_minimum_members = 5
fallback_to_default = true

[sdk_api]
socket_path = "/var/run/bgpx/sdk.sock"
tcp_listen = ""
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/sdk_auth_token"

[routing_policy]
policy_file = "/etc/bgpx/routing_policy.toml"
default_action = "bgpx"

[control_api]
socket_path = "/var/run/bgpx/control.sock"
socket_permissions = "0600"

[reputation]
geo_plausibility_enabled = true
geo_plausibility_warn_threshold = 2.0
geo_plausibility_alert_threshold = 3.0
enforce_global_blacklist = false
blacklist_default_duration_days = 30
blacklist_auto_review = true

[pluggable_transport]
enabled = false
mode = "builtin"
binary_path = ""
listen_addr = "127.0.0.1"
listen_port = 7475
timing_jitter_max_ms = 50
max_padding_bytes = 200
apply_to_mesh = false

[mesh]
enabled = false
transports = []
wifi_mesh_interface = ""
lora_interface = ""
lora_frequency_mhz = 868.0
beacon_interval_seconds = 30

[extensions]
cover_traffic_enabled = false
cover_traffic_target_kbps = 100
path_quality_reporting_enabled = true
cross_domain_routing_enabled = true
domain_bridge_enabled = true

[store_and_forward]
enabled = false
buffer_size_kb = 1024
packet_ttl_seconds = 3600

[metrics]
prometheus_enabled = false
prometheus_addr = "127.0.0.1"
prometheus_port = 9474
```

---

## 4. Logging Policy

Valid production log levels: `error`, `warn`, `info`. `debug` and `trace` are NOT valid in production builds.

### Prohibited Log Content

The following MUST NEVER appear in any log file under any log level:
- Client IP addresses at relay or exit nodes
- Destination addresses at any node
- path_id values
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
2.  Load node identity (Ed25519 private key, derive NodeID)
3.  Initialize data directory; load reputation database and DHT routing table
4.  Initialize Network I/O subsystem (UDP socket, bind port 7474)
5.  Initialize Session manager
6.  Initialize Pool manager; load configured pools; verify signatures
7.  Initialize Domain Manager
    a. For each [[routing_domains]] entry: initialize appropriate transport
    b. If bridge_capable (2+ domains): initialize domain bridge manager
    c. If mesh domains: initialize mesh island manager
8.  Initialize DHT subsystem
    a. Internet mode: connect to bootstrap nodes; populate routing table
    b. Mesh-only mode: broadcast MESH_BEACON; populate from beacons
    c. Publish node advertisement
    d. If bridge_capable: publish DOMAIN_ADVERTISE for each bridge pair
    e. If island manager: publish MESH_ISLAND_ADVERTISE if internet available
9.  Initialize Exit policy engine (if exit/gateway role)
10. Initialize Routing Policy Engine (dual-stack mode)
11. Initialize Pluggable transport (if enabled)
12. Initialize SDK Socket server
13. Initialize Control API server
14. Initialize Metrics exporter (if enabled)
15. Initialize Session re-handshake scheduler (24hr background task)
16. Enter ACTIVE state
    → Log "Node active. NodeID: <node_id>"
```

---

## 6. Graceful Shutdown (SIGTERM/SIGINT)

```
1. Transition to DRAIN state
   a. Publish NODE_WITHDRAW
   b. Set bridge.available = false in DOMAIN_ADVERTISE and re-publish
   c. Wait up to 30 seconds for DHT propagation

2. Stop accepting new sessions

3. Wait for in-flight sessions to complete (up to shutdown_timeout, default 30s)

4. Flush reputation database to disk
5. Flush DHT routing table to disk
6. Remove Control API and SDK socket files

7. Zeroize all session keys in memory (MANDATORY)
   - session_key for all active sessions
   - keepalive_key for all active sessions

8. Exit with code 0
```

---

## 7. SIGHUP (Config Reload)

Live-reloadable:
```
max_sessions, total_bandwidth_mbps, max_session_bandwidth_mbps,
log_level, cover_traffic_enabled, cover_traffic_target_kbps,
exit_policy.allow_ports (new streams), exit_policy.deny_destinations (new streams),
pool discovery configuration, routing policy rules,
domain_bridge bridge availability states,
mesh_island.advertisement_interval_hours
```

Requires restart:
```
private_key_path, listen_addr, listen_port, data_dir,
control_api.socket_path, sdk_api.socket_path,
[[routing_domains]] entries
```

---

## 8. Resource Requirements

| Resource | Relay Only | Domain Bridge (2 domains) |
|---|---|---|
| RAM | 256 MB minimum | 512 MB minimum |
| CPU | 1 core | 2 cores |
| Disk | 1 GB | 2 GB |
| Network | 10 Mbps symmetric relay / 100 Mbps gateway | Same, dual interface |

Memory scaling:
```
estimated_MB ≈ 50 + (session_count × 0.256) + (cross_domain_path_entries × 0.1)
```

---

## 9. Security Hardening

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

---

## 10. Monitoring

New Prometheus metrics for cross-domain:

```
bgpx_routing_domains_active          # Count of active routing domains
bgpx_domain_bridge_transitions_total # Total domain bridge transitions forwarded
bgpx_cross_domain_paths_active       # Active cross-domain paths
bgpx_mesh_island_reachable{island}   # Boolean (0/1) per island
bgpx_bridge_pairs_active             # Count of active bridge pairs
bgpx_cross_domain_path_table_size    # Entries in cross-domain path table
bgpx_bridge_availability{from,to}    # Bridge pair availability (0/1)
```

All existing metrics retained.

---

## 11. Deployment Checklist

Before going live:
- [ ] Node private key generated and securely backed up
- [ ] Configuration complete and validated (`bgpx-node --config-check`)
- [ ] Firewall rules applied (UDP 7474 open)
- [ ] systemd service installed with hardening
- [ ] Public IP and port reachable from internet
- [ ] If exit node: exit policy reviewed, operator contact set, DoH configured, ECH tested
- [ ] If bridge node: both routing_domains configured, bridge pair validated
- [ ] If mesh island gateway: island_id chosen (unique), MESH_ISLAND_ADVERTISE publishing verified
- [ ] Pool memberships configured (if applicable)
- [ ] Prometheus monitoring configured
- [ ] Log rotation configured
- [ ] Backup procedure tested for `/var/lib/bgpx/`
- [ ] Pool curator key backed up separately (if pool curator)
