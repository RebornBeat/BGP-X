# BGP-X Node Daemon Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X node daemon (`bgpx-node`) — the software that runs on relay, entry, exit, and discovery nodes. It covers the daemon's architecture, configuration, lifecycle, operational requirements, and behavior under all conditions.

---

## 1. Overview

The BGP-X node daemon is the core server-side component of the BGP-X overlay network. It is responsible for:

- Accepting and managing BGP-X sessions from clients and other nodes
- Relaying onion-encrypted packets through the overlay
- Participating in the DHT for node discovery
- Publishing and maintaining the node's signed advertisement
- Enforcing exit policy (for exit nodes)
- Exposing a local Control API for management tooling

The daemon is designed to run as a long-lived background process on Linux systems, from commodity VPS instances to embedded router hardware.

---

## 2. Daemon Architecture

The node daemon is structured as a set of cooperating async tasks within a single process. The reference implementation uses Rust with Tokio as the async runtime.

```
bgpx-node process
├── Main task
│   ├── Signal handler (SIGTERM, SIGINT, SIGHUP)
│   └── Lifecycle coordinator
│
├── Network I/O subsystem
│   ├── UDP listener (forwarding pipeline)
│   │   ├── Receiver threads (SO_REUSEPORT, N = CPU cores / 2)
│   │   └── Sender thread pool (4 threads)
│   └── TCP listener (exit node only, outbound connections)
│
├── Session manager
│   ├── Session table (sharded concurrent hash map)
│   ├── Handshake handler
│   └── Keepalive monitor
│
├── DHT subsystem
│   ├── Routing table (k-bucket structure)
│   ├── Advertisement publisher
│   ├── Advertisement store (records this node is responsible for)
│   └── Bootstrap manager
│
├── Control API server
│   └── Unix domain socket (JSON-RPC 2.0)
│
├── Reputation system
│   ├── Reputation database
│   └── Event processor
│
├── Exit policy engine (exit nodes only)
│   ├── Policy evaluator
│   └── Outbound connection manager
│
└── Metrics collector
    ├── In-memory counters
    └── Prometheus exporter (optional)
```

---

## 3. Configuration

The node daemon is configured via a TOML configuration file, by default located at:

```
/etc/bgpx/node.toml        (system install)
~/.config/bgpx/node.toml   (user install)
```

### 3.1 Complete configuration reference

```toml
# BGP-X Node Configuration
# All values shown are defaults unless marked REQUIRED

[node]
# REQUIRED: Path to the node's Ed25519 private key file (PEM or raw bytes)
private_key_path = "/etc/bgpx/node_private_key"

# REQUIRED: Roles this node serves
# Valid values: "relay", "entry", "exit", "discovery"
roles = ["relay", "entry", "discovery"]

# Human-readable node name (used in logs only, not published)
name = "bgpx-node-1"

# Data directory for persistent state (reputation DB, DHT routing table)
data_dir = "/var/lib/bgpx"

# Log level: "error", "warn", "info", "debug", "trace"
log_level = "info"

# Log format: "text" or "json"
log_format = "text"

# Log destination: "stdout", "syslog", or a file path
log_destination = "stdout"

[network]
# UDP listen address and port for BGP-X overlay traffic
listen_addr = "0.0.0.0"
listen_port = 7474

# IPv6 listen address (optional; enables dual-stack)
listen_addr_v6 = "::"
listen_port_v6 = 7474

# Public IP address to advertise in DHT (REQUIRED if behind NAT)
# If not set, the daemon attempts auto-detection via STUN
public_addr = ""

# Public port to advertise (defaults to listen_port)
public_port = 7474

# SO_RCVBUF size in bytes
socket_recv_buffer = 16777216  # 16 MB

# SO_SNDBUF size in bytes
socket_send_buffer = 16777216  # 16 MB

[sessions]
# Maximum concurrent sessions
max_sessions = 10000

# Session idle timeout in seconds (no traffic for this duration = close)
session_idle_timeout_seconds = 90

# Keepalive interval in seconds (randomized ± 5s)
keepalive_interval_seconds = 25

# Maximum handshake completion time in seconds
handshake_timeout_seconds = 10

# Maximum per-session bandwidth in Mbps (0 = unlimited)
max_session_bandwidth_mbps = 0

[bandwidth]
# Total outbound bandwidth cap in Mbps (0 = unlimited)
total_bandwidth_mbps = 0

# Bandwidth reserved for DHT and control traffic (Mbps)
reserved_bandwidth_mbps = 10

[dht]
# Bootstrap nodes (list of "node_id@ip:port" entries)
# These are the well-known BGP-X bootstrap nodes
bootstrap_nodes = [
    "a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1@203.0.113.1:7474",
    "b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3@203.0.113.2:7474",
]

# k value for k-bucket routing table (nodes per bucket)
k = 20

# Alpha value for parallel lookups
alpha = 3

# Routing table refresh interval in minutes
refresh_interval_minutes = 60

# Advertisement re-publication interval in hours
republish_interval_hours = 12

# Advertisement validity period in hours (max 48)
advertisement_validity_hours = 24

# Maximum DHT records this node will store for others
max_stored_records = 10000

[advertisement]
# Self-reported performance metrics (used in node advertisement)
bandwidth_mbps = 100
latency_ms = 12

# Region: "NA", "EU", "AP", "SA", "AF", "ME"
region = "EU"

# Country: ISO 3166-1 alpha-2
country = "DE"

# ASN of the network this node is on
asn = 12345

# Operator public key (base64url Ed25519 public key)
# Links this node to an operator identity for reputation aggregation
operator_id = ""

# Path to operator private key (used to sign exit policies)
# Only required for exit nodes
operator_private_key_path = ""

[exit_policy]
# Only relevant if "exit" is in node.roles

# Protocols to forward: ["tcp"], ["udp"], or ["tcp", "udp"]
allow_protocols = ["tcp", "udp"]

# Destination ports to allow (empty list = all ports)
# Recommended: restrict to common service ports
allow_ports = [80, 443, 8080, 8443, 8000]

# CIDR ranges to always deny (private ranges MUST be included)
deny_destinations = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
]

# Logging policy: "none", "metadata", "full"
logging_policy = "none"

# Operator contact (published in exit policy)
operator_contact = ""

# Jurisdiction (ISO 3166-1 country code of legal jurisdiction)
jurisdiction = "DE"

# Maximum outbound connections at the exit node
max_outbound_connections = 2000

# Connection timeout for outbound clearnet connections (seconds)
outbound_connect_timeout_seconds = 10

[control_api]
# Unix socket path for Control API
socket_path = "/var/run/bgpx/control.sock"

# Socket file permissions (octal)
socket_permissions = "0600"

[reputation]
# Enable distributed reputation system (publish/receive reputation records)
distributed_reputation_enabled = false

# Minimum reporter reputation to accept distributed reports
min_reporter_reputation = 70.0

# Blacklist threshold (reputation below this = blacklisted, after min_observations)
blacklist_threshold = 15.0

# Minimum observations before blacklisting
min_observations_before_blacklist = 10

[extensions]
# Enable cover traffic generation
cover_traffic_enabled = false

# Cover traffic target rate in Kbps
cover_traffic_target_kbps = 100

# Enable pluggable transport (obfuscates BGP-X traffic)
pluggable_transport_enabled = false
pluggable_transport_binary = ""

# Enable stream multiplexing
multiplexing_enabled = true

# Enable path quality reporting
path_quality_reporting_enabled = true

[metrics]
# Enable Prometheus metrics exporter
prometheus_enabled = false

# Prometheus listen address
prometheus_addr = "127.0.0.1"
prometheus_port = 9474
```

---

## 4. Node Lifecycle

### 4.1 Startup sequence

```
1. Parse and validate configuration file
   ├── Verify required fields are present
   ├── Verify private key file is readable and valid
   └── If exit role: verify exit policy fields are complete

2. Load or generate node identity
   ├── Load Ed25519 private key from private_key_path
   ├── Derive public key and NodeID
   └── Log NodeID at INFO level

3. Initialize data directory
   ├── Create data_dir if not exists
   ├── Load reputation database from data_dir/reputation.db
   └── Load DHT routing table from data_dir/routing_table.db

4. Initialize network subsystem
   ├── Bind UDP socket to listen_addr:listen_port
   ├── Set socket buffer sizes
   └── If IPv6 enabled: bind IPv6 socket

5. Initialize session manager
   └── Start keepalive monitor background task

6. Initialize DHT subsystem
   ├── Load routing table from disk
   ├── Connect to bootstrap nodes
   ├── Perform DHT_FIND_NODE(own NodeID) to populate routing table
   └── On success: publish node advertisement (DHT_PUT)

7. Initialize exit policy engine (if exit role)
   ├── Load exit policy from configuration
   ├── Sign exit policy with operator private key
   └── Include signed policy in node advertisement

8. Start Control API server
   └── Bind Unix socket at control_api.socket_path

9. Start metrics exporter (if enabled)

10. Enter ACTIVE state
    └── Log "Node active. NodeID: <node_id>" at INFO level
```

### 4.2 SIGHUP handling (config reload)

On receiving SIGHUP, the daemon reloads the configuration file. Live-reloadable options are applied immediately. Non-reloadable options require a full restart.

Live-reloadable options:
- `max_sessions`
- `total_bandwidth_mbps`
- `max_session_bandwidth_mbps`
- `log_level`
- `cover_traffic_enabled`
- `cover_traffic_target_kbps`
- `exit_policy.allow_ports` (takes effect for new streams only)
- `exit_policy.deny_destinations` (takes effect for new streams only)

Non-reloadable (require restart):
- `private_key_path`
- `listen_addr` / `listen_port`
- `data_dir`
- `control_api.socket_path`

### 4.3 Graceful shutdown (SIGTERM / SIGINT)

```
1. Transition to DRAIN state
   └── Stop accepting new sessions (reject new HANDSHAKE_INIT)

2. Wait for in-flight sessions to complete
   ├── Send KEEPALIVE to all active sessions
   ├── Wait up to shutdown_timeout_seconds (default 30)
   └── After timeout: force-close remaining sessions

3. Flush reputation database to disk

4. Flush DHT routing table to disk

5. Remove Control API socket file

6. Zeroize all session keys in memory

7. Exit with code 0
```

### 4.4 Crash recovery

On unexpected termination (crash, OOM kill, power loss):

- All in-flight sessions are irrecoverably lost (no session resumption by design)
- The DHT routing table and reputation database are restored from disk on next start
- The node re-publishes its advertisement within the bootstrap phase
- Clients whose paths included this node will detect the failure via KEEPALIVE timeout and rebuild their paths

---

## 5. Key Management

### 5.1 Key generation

The node's Ed25519 private key is generated once and stored at `private_key_path`. It MUST NOT be regenerated unless the operator intentionally changes the node's identity (doing so changes the NodeID and loses all accumulated reputation).

Key generation command (provided by `bgpx-cli`):

```bash
bgpx-cli keygen --output /etc/bgpx/node_private_key
```

Output format: raw 32-byte Ed25519 seed, written to a file with permissions 0600.

### 5.2 Key storage requirements

The private key file MUST be:
- Readable only by the bgpx-node process user (permissions 0600)
- Stored on a filesystem that does not make temporary copies (avoid `/tmp` and network filesystems)
- Backed up securely (loss of the private key means loss of node identity and reputation)

For high-security deployments, the key SHOULD be stored in a hardware security module (HSM) or a secrets manager (e.g., HashiCorp Vault). HSM integration is a planned extension.

### 5.3 Key rotation

Rotating the node private key changes the NodeID and requires:
1. Generating a new keypair
2. Publishing a signed rotation notice from the old key (planned feature)
3. Updating the `private_key_path` configuration
4. Restarting the daemon

Until the rotation notice feature is implemented, key rotation is equivalent to establishing a new node identity. Existing reputation is not transferable.

---

## 6. Resource Requirements

### 6.1 Minimum hardware (relay node)

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 1 core | 4 cores |
| RAM | 256 MB | 1 GB |
| Disk | 1 GB | 10 GB |
| Network | 10 Mbps symmetric | 1 Gbps symmetric |
| OS | Linux kernel 5.15+ | Linux kernel 6.1+ |

### 6.2 Minimum hardware (exit / gateway node)

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores | 8 cores |
| RAM | 512 MB | 4 GB |
| Disk | 5 GB | 20 GB |
| Network | 100 Mbps symmetric | 10 Gbps symmetric |
| OS | Linux kernel 5.15+ | Linux kernel 6.1+ |

Exit nodes require more resources because they:
- Maintain outbound TCP connections to clearnet destinations (each connection uses OS resources)
- Perform DNS resolution
- Handle response buffering for clearnet connections

### 6.3 Memory scaling

Memory usage scales approximately linearly with session count:

```
estimated_memory_MB ≈ base_overhead_MB + (session_count × 0.256 MB)
                    ≈ 50 + (10000 × 0.256)
                    ≈ 2610 MB at maximum sessions (10,000)
```

For nodes with fewer than 2,000 concurrent sessions, 512 MB RAM is sufficient.

---

## 7. Security Hardening

The following hardening measures SHOULD be applied to production node deployments:

### 7.1 Process isolation

```bash
# Run as a dedicated non-root user
useradd --system --no-create-home --shell /sbin/nologin bgpx

# Set file ownership
chown bgpx:bgpx /etc/bgpx/node_private_key
chown -R bgpx:bgpx /var/lib/bgpx
chown bgpx:bgpx /var/run/bgpx
```

### 7.2 systemd service hardening

```ini
[Service]
User=bgpx
Group=bgpx
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/bgpx /var/run/bgpx
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
SystemCallFilter=@system-service
MemoryDenyWriteExecute=true
RestrictNamespaces=true
RestrictRealtime=true
LockPersonality=true
```

### 7.3 Network firewall rules

For a relay-only node (no exit), the following iptables rules apply:

```bash
# Allow BGP-X overlay traffic
iptables -A INPUT -p udp --dport 7474 -j ACCEPT

# Allow Control API (local only — no inbound rule needed; Unix socket)

# Drop all other inbound traffic
iptables -A INPUT -j DROP

# Allow all outbound BGP-X traffic (to other relay nodes)
iptables -A OUTPUT -p udp --dport 7474 -j ACCEPT

# Allow outbound DHT traffic (same port)
# Allow outbound DNS for bootstrap resolution
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT

# Drop other outbound (relay nodes should not initiate clearnet connections)
iptables -A OUTPUT -j DROP
```

For exit nodes, outbound clearnet traffic must be permitted on relevant ports per exit policy.

### 7.4 Kernel parameters

Recommended sysctl settings:

```ini
# Network performance
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.core.netdev_max_backlog = 65536

# Security
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
```

---

## 8. Monitoring and Observability

### 8.1 Log output

The daemon logs structured events to the configured destination. All log entries include:
- Timestamp (ISO 8601 UTC)
- Log level
- Component (network, dht, session, exit, control_api)
- Message

Log entries MUST NOT include:
- Client IP addresses
- Session IDs
- Destination addresses
- Any traffic content

Example log output (JSON format):

```json
{"timestamp":"2026-04-24T12:00:00Z","level":"INFO","component":"dht","message":"Bootstrap complete. Routing table populated with 342 nodes."}
{"timestamp":"2026-04-24T12:00:01Z","level":"INFO","component":"network","message":"Node advertisement published. Expires in 24 hours."}
{"timestamp":"2026-04-24T12:05:23Z","level":"WARN","component":"session","message":"Keepalive timeout. Session closed."}
```

### 8.2 Prometheus metrics

When `prometheus_enabled = true`, the daemon exposes metrics at:

```
http://prometheus_addr:prometheus_port/metrics
```

Key metrics exposed:

```
# HELP bgpx_sessions_active Current number of active sessions
# TYPE bgpx_sessions_active gauge
bgpx_sessions_active 42

# HELP bgpx_packets_relayed_total Total packets relayed
# TYPE bgpx_packets_relayed_total counter
bgpx_packets_relayed_total 1428571

# HELP bgpx_bytes_relayed_total Total bytes relayed
# TYPE bgpx_bytes_relayed_total counter
bgpx_bytes_relayed_total 10737418240

# HELP bgpx_handshake_total Total handshakes by result
# TYPE bgpx_handshake_total counter
bgpx_handshake_total{result="success"} 423
bgpx_handshake_total{result="timeout"} 2
bgpx_handshake_total{result="invalid"} 0

# HELP bgpx_dht_routing_table_size Current size of DHT routing table
# TYPE bgpx_dht_routing_table_size gauge
bgpx_dht_routing_table_size 342

# HELP bgpx_reputation_blacklisted_nodes Currently blacklisted nodes
# TYPE bgpx_reputation_blacklisted_nodes gauge
bgpx_reputation_blacklisted_nodes 2
```

---

## 9. Operational Procedures

### 9.1 Initial deployment checklist

Before going live, verify:

- [ ] Node private key generated and backed up securely
- [ ] Configuration file complete and validated (`bgpx-node --config-check`)
- [ ] Firewall rules applied
- [ ] systemd service file installed and hardened
- [ ] Public IP and port are reachable from the internet
- [ ] If exit node: exit policy reviewed, operator contact set, exit policy signed
- [ ] If exit node: legal review of exit policy and jurisdiction completed
- [ ] Prometheus monitoring configured (if applicable)
- [ ] Log rotation configured (logrotate or equivalent)
- [ ] Backup procedure for `/var/lib/bgpx/` tested

### 9.2 Advertisement verification

After startup, verify the node advertisement was published successfully:

```bash
bgpx-cli advertisement status
# Output:
# Status: published
# NodeID: a3f2...b9c1
# Roles: relay, entry
# Signed at: 2026-04-24T12:00:00Z
# Expires at: 2026-04-25T12:00:00Z
# DHT storage nodes reached: 18
```

To retrieve the published advertisement from the DHT (as a client would see it):

```bash
bgpx-cli advertisement verify --node-id a3f2...b9c1
# Fetches the advertisement from the DHT and validates the signature
```

### 9.3 Routine maintenance

| Task | Frequency | Command |
|---|---|---|
| Check node status | Daily | `bgpx-cli status` |
| Review error logs | Daily | `journalctl -u bgpx-node --since yesterday` |
| Check reputation stats | Weekly | `bgpx-cli stats --window 168h` |
| Verify advertisement | Weekly | `bgpx-cli advertisement verify --self` |
| Apply software updates | Per release | `systemctl restart bgpx-node` after update |
| Back up key and data | Weekly | Back up `/etc/bgpx/` and `/var/lib/bgpx/` |
