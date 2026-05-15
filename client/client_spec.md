# BGP-X Client Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X control client (`bgpx-cli`) — the command-line tool for managing and monitoring BGP-X node daemons. It is a **configuration and management tool**. It is **NOT** a routing stack.

---

## 1. What the BGP-X Client Is

The BGP-X control client (`bgpx-cli`) is a command-line interface that:

- Connects to a running BGP-X node daemon via the Control Socket
- Sends JSON-RPC 2.0 commands to manage the daemon
- Retrieves status, monitoring, and diagnostic information
- Configures paths, pools, routing policy, and domains
- Manages node identity and advertisements

The client is a **management tool only**. All routing, encryption, DHT participation, and network operations are performed by the **bgpx-node daemon** — specified in `/node/node.md`.

---

## 2. What the BGP-X Client Is NOT

The control client is **NOT**:

- A routing stack (that is `bgpx-node`)
- An overlay network daemon
- A TUN interface creator
- An onion encryption engine
- A DHT participant
- A session manager
- A path constructor

All of the above are performed by the **bgpx-node daemon**. The client sends commands to the daemon; the daemon performs the work.

---

## 3. Client Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      bgpx-cli Process                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Command Parser                         │   │
│  │      Parses arguments, validates syntax                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                JSON-RPC 2.0 Client                       │   │
│  │   Constructs request objects, handles responses         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Control Socket Client                       │   │
│  │   Unix domain socket connection to daemon               │   │
│  │   /var/run/bgpx/control.sock (default)                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
└───────────────────────────│─────────────────────────────────────┘
                            │
                   Unix Domain Socket
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                       bgpx-node Daemon                            │
│                        (Control API Server)                       │
│                                                                   │
│  - Routing engine                                                 │
│  - Onion encryption                                               │
│  - DHT participation                                              │
│  - Session management                                             │
│  - Path construction                                              │
│  - TUN interface                                                  │
│  - All network operations                                         │
└───────────────────────────────────────────────────────────────────┘
```

---

## 4. Connection Model

### 4.1 Default Connection

The client connects to the daemon's Control Socket:

```
/var/run/bgpx/control.sock      (default, system daemon)
~/.bgpx/control.sock            (user daemon)
```

The socket is a Unix domain socket with JSON-RPC 2.0 protocol.

### 4.2 Remote Connection (via SSH Tunnel)

For managing a daemon on a remote router:

```bash
# SSH tunnel to forward control socket
ssh -L /tmp/bgpx-remote.sock:/var/run/bgpx/control.sock user@router

# Connect through tunnel
bgpx-cli --socket /tmp/bgpx-remote.sock node show
```

### 4.3 TCP Connection (Optional)

The daemon can be configured to expose a TCP endpoint for the Control API:

```toml
[control_api]
tcp_listen = "127.0.0.1:7476"   # localhost only for security
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/control_token"
```

TCP mode requires authentication via token. Use only on trusted networks or with VPN.

---

## 5. CLI Interface

### 5.1 General Syntax

```bash
bgpx-cli [global-options] <command> [command-options] [arguments]
```

### 5.2 Global Options

| Option | Description |
|---|---|
| `--socket <path>` | Path to control socket (default: `/var/run/bgpx/control.sock`) |
| `--timeout <seconds>` | Request timeout (default: 30) |
| `--format <json|text>` | Output format (default: text) |
| `--quiet` | Suppress non-essential output |
| `--verbose` | Verbose output |
| `--help` | Show help |
| `--version` | Show version |

---

## 6. Command Categories

### 6.1 Node Management

```bash
# Show node status
bgpx-cli node show

# Show node identity
bgpx-cli node identity

# Show node statistics
bgpx-cli node stats

# Ping a node by NodeID
bgpx-cli node ping <node_id> [--domain <domain>]

# Withdraw node from network
bgpx-cli node withdraw [--old-key <path>]

# Show DHT statistics
bgpx-cli node dht-stats

# Show recent events
bgpx-cli node events --last <duration>
```

### 6.2 Session Management

```bash
# List active sessions
bgpx-cli sessions list

# Close a session
bgpx-cli sessions close <session_id>
```

### 6.3 Path Management

```bash
# List active paths
bgpx-cli paths list

# Build a new path
bgpx-cli paths build <destination> [--domain-segments <json>]

# Test path to destination
bgpx-cli paths test <destination>

# Rebuild a specific path
bgpx-cli paths rebuild <path_id>
```

### 6.4 Pool Management

```bash
# List known pools
bgpx-cli pools list

# Add a pool
bgpx-cli pools add --pool <json>

# Remove a pool
bgpx-cli pools remove <pool_id>

# Refresh pool membership
bgpx-cli pools refresh <pool_id>
```

### 6.5 Routing Policy Management

```bash
# List routing policy rules
bgpx-cli policy list

# Add a routing rule
bgpx-cli policy add --rule <json> [--position <n>]

# Remove a rule
bgpx-cli policy remove <rule_id>

# Test which rule matches a destination
bgpx-cli policy test --destination <dest> [--device-ip <ip>]

# Reload policy from file
bgpx-cli policy reload
```

### 6.6 Domain Management

```bash
# List routing domains
bgpx-cli domains list

# Show domain details
bgpx-cli domains show <domain_id>

# List domain bridge nodes
bgpx-cli domains bridges [--self] [--health] [--latency-report]

# Test connectivity to a domain
bgpx-cli domains test-connectivity <domain>

# Discover bridges between domains
bgpx-cli domains discover-bridges --from <domain> --to <domain>

# Update bridge availability
bgpx-cli domains bridge-availability --from <domain> --to <domain> --available <bool>
```

### 6.7 Mesh Island Management

```bash
# List mesh islands
bgpx-cli islands list [--global] [--dht]

# Show island details
bgpx-cli islands show <island_id> [--dht-freshness] [--dht-global]

# Publish island advertisement
bgpx-cli islands publish <island_id>

# Withdraw island advertisement
bgpx-cli islands withdraw <island_id>

# Test island health
bgpx-cli islands health <island_id>

# Test connectivity to an island
bgpx-cli islands connectivity-test <island_id> [--from-clearnet]

# Show bridges for an island
bgpx-cli islands bridges <island_id>
```

### 6.8 Node Database Management

```bash
# Blacklist a node
bgpx-cli nodes blacklist <node_id> --reason <text>

# Unblacklist a node
bgpx-cli nodes unblacklist <node_id>

# Show node reputation
bgpx-cli nodes reputation <node_id>
```

### 6.9 Reputation Management

```bash
# Show local reputation scores
bgpx-cli reputation show [--self]

# Show recent reputation events
bgpx-cli reputation events [--last <duration>] [--domain <domain>]
```

### 6.10 Exit Policy Management

```bash
# Show exit policy
bgpx-cli exit-policy show [--version]

# Reload exit policy from file
bgpx-cli exit-policy reload
```

### 6.11 Name Registry Management

```bash
# Register a name
bgpx-cli names register <name> --service <service_id>

# Lookup a name
bgpx-cli names lookup <name>

# List registered names
bgpx-cli names list [--self]

# Withdraw a name
bgpx-cli names withdraw <name>
```

### 6.12 Certification Management

```bash
# Apply for node certification
bgpx-cli cert apply --tier <1|2|3|4>

# Show certification status
bgpx-cli cert show [--self]
```

### 6.13 Event Subscription

```bash
# Subscribe to events (streaming)
bgpx-cli events subscribe [--types <list>]

# Show last N events
bgpx-cli events last [--n <count>] [--domain <domain>]
```

---

## 7. Output Formats

### 7.1 Text Format (Default)

Human-readable output with tables and indented structures:

```
$ bgpx-cli node show

Node Status
──────────────────────────────────────────
Node ID:        a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
Status:         ACTIVE
Uptime:         5d 14h 23m
Sessions:       47 active
Paths:          12 (3 pre-built)
Pool Memberships: bgpx-default, eu-exit-pool

Routing Domains
──────────────────────────────────────────
  clearnet (0x00000001)
    Endpoints: 203.0.113.1:7474
    
  mesh:lima-district-1 (0x00000003-a1b2c3d4)
    Transports: WiFi mesh, LoRa
    
Bridge Capable: Yes
  clearnet ↔ mesh:lima-district-1 (latency: 25ms)
```

### 7.2 JSON Format

Structured output for scripting:

```bash
$ bgpx-cli --format json node show
```

```json
{
  "node_id": "a3f2b9c1...",
  "status": "ACTIVE",
  "uptime_seconds": 4752180,
  "sessions": {
    "active": 47
  },
  "paths": {
    "active": 12,
    "pre_built": 3
  },
  "pool_memberships": ["bgpx-default", "eu-exit-pool"],
  "routing_domains": [
    {
      "domain_type": "clearnet",
      "domain_id": "0x00000001-00000000",
      "endpoints": [{"protocol": "udp", "address": "203.0.113.1", "port": 7474}]
    },
    {
      "domain_type": "mesh",
      "domain_id": "0x00000003-a1b2c3d4",
      "island_id": "lima-district-1",
      "transports": ["wifi_mesh", "lora"]
    }
  ],
  "bridge_capable": true,
  "bridges": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "latency_ms": 25
    }
  ]
}
```

---

## 8. Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 2 | Invalid arguments |
| 3 | Connection failed |
| 4 | Timeout |
| 5 | Permission denied |
| 6 | Not found |
| 7 | Daemon error |

---

## 9. Configuration Client vs SDK Application

| Aspect | Configuration Client | SDK Application |
|---|---|---|
| **Purpose** | Manage and monitor daemon | Build BGP-X-aware applications |
| **Connection** | Control Socket | SDK Socket |
| **Protocol** | JSON-RPC 2.0 (text) | JSON-RPC 2.0 with binary extensions |
| **Capabilities** | Status, config, management | Open streams, register services, path control |
| **Typical user** | Operator / Administrator | Application developer |

The SDK is documented in `/sdk/sdk_spec.md`. The SDK connects to the SDK Socket (`/var/run/bgpx/sdk.sock`) and provides application-level functionality.

---

## 10. Security Model

### 10.1 Socket Permissions

The Control Socket should have restrictive permissions:

```
srwx------ 1 root root 0 Apr 24 12:00 /var/run/bgpx/control.sock
```

Only the `root` user or users in the `bgpx` group should be able to connect.

### 10.2 Authentication for TCP Mode

If TCP mode is enabled:

```toml
[control_api]
tcp_listen = "127.0.0.1:7476"
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/control_token"
```

The auth token file contains a random 32-byte token (hex-encoded):

```
# Generate token
openssl rand -hex 32 > /etc/bgpx/control_token
chmod 600 /etc/bgpx/control_token
```

Client usage:

```bash
bgpx-cli --tcp 127.0.0.1:7476 --token /path/to/token node show
```

### 10.3 No Logging of Sensitive Data

The client itself does not log sensitive information. All logging is performed by the daemon, which follows the strict no-log policy defined in the architecture.

---

## 11. Platform Support

| Platform | Support Level |
|---|---|
| Linux (x86_64, ARM) | Full |
| macOS (x86_64, ARM) | Full |
| Windows (x86_64) | Planned |
| Android | Planned (via Termux or dedicated app) |
| iOS | Not applicable (no terminal) |

---

## 12. GUI Alternatives

### 12.1 LuCI Web Interface

For OpenWrt routers, the BGP-X LuCI package provides a web-based management interface at `http://192.168.1.1/cgi-bin/luci/admin/bgpx/`.

### 12.2 Planned Desktop GUI

A cross-platform desktop application (Electron-based) is planned. It will use the same JSON-RPC Control API as the CLI.

---

## 13. Example Usage Scenarios

### 13.1 Check Node Status

```bash
$ bgpx-cli node show
```

### 13.2 Build a Cross-Domain Path

```bash
$ bgpx-cli paths build example.com \
    --domain-segments '[
        {"type": "segment", "domain": "clearnet", "hops": 2},
        {"type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:lima-district-1"},
        {"type": "segment", "domain": "mesh:lima-district-1", "hops": 2}
    ]'
```

### 13.3 Register a BGP-X Native Service

```bash
$ bgpx-cli names register myservice --service a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

### 13.4 Monitor Active Paths

```bash
$ bgpx-cli paths list --format json | jq '.[] | select(.quality_score < 0.5)'
```

### 13.5 Add a Routing Policy Rule

```bash
$ bgpx-cli policy add --rule '{
    "priority": 10,
    "match": {"destination_domain": ["*.sensitive.org"]},
    "action": "bgpx",
    "path_constraints": {
        "min_hops": 5,
        "require_ech": true,
        "exit_jurisdiction_blacklist": ["US", "GB"]
    }
}'
```

---

## 14. Relationship to Other Components

| Document | Relationship |
|---|---|
| `/node/node.md` | Daemon that this client controls |
| `/node/api.md` | Internal API the daemon exposes |
| `/sdk/sdk_spec.md` | SDK for building BGP-X-aware applications |
| `/control-plane/control_api.md` | JSON-RPC API specification |

---

## 15. Configuration File Location

The client itself does not use a configuration file. All connection parameters are specified via command-line arguments or environment variables:

```bash
# Environment variables
export BGPX_CONTROL_SOCKET=/var/run/bgpx/control.sock
export BGPX_TIMEOUT=60
export BGPX_FORMAT=json

bgpx-cli node show   # Uses environment variables
```

---

## 16. Summary

The BGP-X control client (`bgpx-cli`) is:

- A **management tool** for bgpx-node daemons
- A **JSON-RPC 2.0 client** connecting to the Control Socket
- A **CLI interface** for configuration, monitoring, and diagnostics
- **NOT a routing stack** — that is bgpx-node

For building BGP-X-aware applications, use the SDK (`/sdk/sdk_spec.md`).

For the daemon specification, see `/node/node.md`.
