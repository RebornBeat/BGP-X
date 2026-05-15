# BGP-X Control Client Specification

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

This document specifies the BGP-X control client (`bgpx-cli` and GUI alternatives) — the management tool for a running BGP-X daemon.

---

## 1. What the Control Client Is

The BGP-X control client is a **configuration and management tool**. It is NOT a routing stack.

The control client:
- Connects to a running BGP-X daemon via the Control Socket
- Reads and writes daemon configuration
- Displays status, statistics, and path information
- Manages routing policy rules
- Manages pools
- Manages node reputation
- Triggers operational actions (path rebuild, advertisement re-publish, etc.)
- Monitors daemon events
- Manages domains, islands, and domain bridges

The control client does NOT:
- Route any traffic
- Construct paths (it can request paths, but the daemon constructs them)
- Perform cryptographic handshakes
- Maintain a DHT connection
- Encrypt or decrypt anything
- Create TUN interfaces
- Perform onion encryption
- Manage sessions

---

## 2. What the Control Client Is NOT

The control client is **NOT**:

- A routing stack (that is `bgpx-node`, specified in `/node/node.md`)
- An overlay network daemon
- A TUN interface creator
- An onion encryption engine
- A DHT participant
- A session manager
- A path constructor

All of the above are performed by the **bgpx-node daemon**. The client sends commands to the daemon; the daemon performs the work.

---

## 3. Architecture

```
[bgpx-cli or GUI]
        │
        │ JSON-RPC 2.0 over Unix socket
        │ (or SSH tunnel from remote machine)
        ▼
[BGP-X Daemon Control Socket]
/var/run/bgpx/control.sock
```

The daemon runs independently of the control client. The daemon continues operating when the control client is not connected. The control client is a management interface, not a required runtime component.

### Client Architecture Diagram

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

## 4. Connection

### 4.1 Local (on same machine as daemon)

```bash
bgpx-cli status
# Automatically connects to /var/run/bgpx/control.sock
```

The client tries sockets in this order:
1. `--socket <path>` argument (if provided)
2. `BGPX_CONTROL_SOCKET` environment variable
3. `/var/run/bgpx/control.sock` (system daemon)
4. `~/.bgpx/control.sock` (user daemon)

### 4.2 Remote (via SSH tunnel)

```bash
# Terminal 1: Forward control socket
ssh -L /tmp/bgpx-remote.sock:/var/run/bgpx/control.sock user@router

# Terminal 2: Use forwarded socket
BGPX_CONTROL_SOCKET=/tmp/bgpx-remote.sock bgpx-cli status
```

Or using the `--socket` flag:

```bash
bgpx-cli --socket /tmp/bgpx-remote.sock status
```

### 4.3 Configuration File

```toml
# ~/.config/bgpx/cli.toml

[connection]
socket_path = "/var/run/bgpx/control.sock"
# or
# socket_path = "/tmp/bgpx-remote.sock"  # SSH tunnel

[output]
format = "text"  # text, json
quiet = false
verbose = false

[timeouts]
default = 30  # seconds
```

### 4.4 TCP Connection (Optional)

The daemon can be configured to expose a TCP endpoint for the Control API:

```toml
[control_api]
tcp_listen = "127.0.0.1:7476"   # localhost only for security
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/control_token"
```

TCP mode requires authentication via token. Use only on trusted networks or with VPN.

Client usage with TCP:

```bash
bgpx-cli --tcp 127.0.0.1:7476 --token /path/to/token node show
```

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
| `--tcp <addr:port>` | TCP endpoint instead of Unix socket |
| `--token <path>` | Auth token file for TCP connection |
| `--timeout <seconds>` | Request timeout (default: 30) |
| `--format <json|text>` | Output format (default: text) |
| `--quiet` | Suppress non-essential output |
| `--verbose` | Verbose output |
| `--help` | Show help |
| `--version` | Show version |

---

## 6. Command Reference

### 6.1 Node Lifecycle

```bash
# Show node status and statistics
bgpx-cli status

# Live statistics over time window
bgpx-cli stats --window 1h

# Watch live events (streaming)
bgpx-cli events watch

# DHT statistics
bgpx-cli dht stats

# Graceful shutdown
bgpx-cli shutdown [--timeout 30] [--no-withdraw]

# Reload configuration
bgpx-cli reload-config
```

### 6.2 Node Management

```bash
# Show node status
bgpx-cli node show

# Show node identity
bgpx-cli node identity

# Show node statistics
bgpx-cli node stats

# Ping a node by NodeID
bgpx-cli node ping <node_id> [--domain <domain>]

# Withdraw node from network (graceful shutdown)
bgpx-cli node withdraw [--old-key <path>]

# Show DHT statistics
bgpx-cli node dht-stats

# Show recent events
bgpx-cli node events --last <duration>
```

### 6.3 Session Management

```bash
# List active sessions
bgpx-cli sessions list

# Close a specific session
bgpx-cli sessions close <session_id>

# View sessions due for re-handshake
bgpx-cli sessions rehandshake-status

# Force re-handshake for all eligible sessions
bgpx-cli sessions force-rehandshake
```

### 6.4 Path Management

```bash
# List active paths
bgpx-cli paths list

# Build a new path with pool-based segments
bgpx-cli paths build --segments "bgpx-default:3,private-exits:1"

# Build a cross-domain path
bgpx-cli paths build --domain-segments "clearnet:2,bridge:clearnet→mesh:island-1,mesh:island-1:2"

# Build path to a specific destination
bgpx-cli paths build <destination> [--domain-segments <json>]

# Test path to destination
bgpx-cli paths test --destination "example.com:443"

# Rebuild a specific path
bgpx-cli paths rebuild <path_id>

# Destroy a path
bgpx-cli paths destroy --index 0
```

### 6.5 Pool Management

```bash
# List all pools
bgpx-cli pools list

# List pools with mesh nodes only
bgpx-cli pools list-mesh

# List pools in a specific domain
bgpx-cli pools list [--domain "mesh:island-1"]

# Get pool details
bgpx-cli pools get --pool-id hex...

# Add private pool from file
bgpx-cli pools add --file /etc/bgpx/my-pool.json

# Remove a pool
bgpx-cli pools remove --pool-id hex...

# Verify pool health (signature + member availability)
bgpx-cli pools verify --pool-id hex...

# Force refresh pool member list
bgpx-cli pools refresh --pool-id hex...

# Rotate pool curator key
bgpx-cli pools rotate-key \
    --pool-id hex... \
    --old-key /etc/bgpx/curator_key_old \
    --new-key /etc/bgpx/curator_key_new \
    --reason scheduled
```

### 6.6 Routing Policy Management

```bash
# List all routing policy rules
bgpx-cli policy list

# Add a rule (insert at position 5)
bgpx-cli policy add \
    --position 5 \
    --destination-domain "*.sensitive.com" \
    --action bgpx \
    --hops 6 \
    --require-ech \
    --exit-logging-policy none

# Add a rule from file
bgpx-cli policy add --file /etc/bgpx/rules/rule.toml --position 5

# Remove a rule
bgpx-cli policy remove --rule-id rule-abc123

# Reorder a rule
bgpx-cli policy reorder --rule-id rule-abc123 --position 2

# Test which rule matches
bgpx-cli policy test --destination example.com --device 192.168.1.50

# Test with specific protocol/port
bgpx-cli policy test --destination "example.com" --protocol tcp --port 443

# Reload policy from file
bgpx-cli policy reload

# Set default action
bgpx-cli policy set-default --action bgpx

# Show policy statistics
bgpx-cli policy stats
```

### 6.7 Node Database Management

```bash
# List nodes with filters
bgpx-cli nodes list [--domain "mesh:island-1"] [--bridge-capable] [--bridge-to "clearnet"]

# List nodes by role
bgpx-cli nodes list --role exit --region EU --min-reputation 80 --ech-capable

# Get node details
bgpx-cli nodes get --node-id hex...

# Blacklist a node
bgpx-cli nodes blacklist hex... --reason "suspected MITM" --duration 720h

# Remove from blacklist
bgpx-cli nodes unblacklist hex...

# List blacklisted nodes
bgpx-cli nodes list-blacklisted

# Check geographic plausibility
bgpx-cli geo check --node-id hex...
bgpx-cli geo check --self
```

### 6.8 Reputation Management

```bash
# Show local reputation scores
bgpx-cli reputation show [--self]

# Show recent reputation events
bgpx-cli reputation events [--last <duration>] [--domain <domain>]
```

### 6.9 Domain Management

```bash
# List routing domains
bgpx-cli domains list

# Show domain details
bgpx-cli domains show <domain_id>

# List domain bridge nodes
bgpx-cli domains bridges [--from clearnet] [--to "mesh:island-1"] [--min-reputation 70]

# Show bridges with health status
bgpx-cli domains bridges [--self] [--health] [--latency-report]

# Show bridge DHT freshness
bgpx-cli domains bridges [--dht-freshness]

# Test connectivity to a domain
bgpx-cli domains test --from clearnet --to "mesh:island-1"

# Discover bridges between domains
bgpx-cli domains discover-bridges --from clearnet --to "mesh:island-1"

# Update bridge availability
bgpx-cli domains bridge-availability --from <domain> --to <domain> --available <bool>
```

### 6.10 Mesh Island Management

```bash
# List mesh islands
bgpx-cli islands list [--online-only]

# List islands from global DHT
bgpx-cli islands list --global --dht

# Show island details
bgpx-cli islands show <island_id> [--dht-freshness] [--dht-global]

# Show island status (local gateway perspective)
bgpx-cli domains island-status --island-id island-name

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

### 6.11 Advertisement Management

```bash
# View current node advertisement
bgpx-cli advertisement show

# Force re-publication
bgpx-cli advertisement republish

# Update performance metrics
bgpx-cli advertisement update-metrics --bandwidth 200 --latency 10

# Publish withdrawal (for graceful shutdown)
bgpx-cli advertisement withdraw
```

### 6.12 Exit Node Management (Exit Nodes Only)

```bash
# Test ECH for a destination
bgpx-cli exit test-ech --destination example.com

# View ECH configuration cache
bgpx-cli exit ech-cache

# Clear ECH cache
bgpx-cli exit clear-ech-cache

# Test DNS resolution (DoH)
bgpx-cli exit test-dns --query example.com

# View exit policy
bgpx-cli exit policy show

# Show exit policy version
bgpx-cli exit-policy show [--version]

# Reload exit policy from file
bgpx-cli exit-policy reload
```

### 6.13 Mesh Management

```bash
# List mesh peers
bgpx-cli mesh peers

# Force beacon broadcast
bgpx-cli mesh beacon

# View gateway bridge status
bgpx-cli gateway status

# View DHT cross-domain sync status
bgpx-cli gateway dht-sync-status
```

### 6.14 SDK Socket Configuration

```bash
# View SDK socket config
bgpx-cli sdk-socket status

# Enable TCP endpoint for LAN access
bgpx-cli sdk-socket tcp-enable --addr 192.168.1.1:7475 --auth-required

# Disable TCP endpoint
bgpx-cli sdk-socket tcp-disable
```

### 6.15 Name Registry Management

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

### 6.16 Certification Management

```bash
# Apply for node certification
bgpx-cli cert apply --tier <1|2|3|4>

# Show certification status
bgpx-cli cert show [--self]
```

### 6.17 Event Subscription

```bash
# Subscribe to events (streaming)
bgpx-cli events subscribe [--filter domain_bridge,mesh_island,cross_domain_path]

# Subscribe to all events
bgpx-cli events watch [--filter all]

# Show last N events
bgpx-cli events last [--n <count>] [--domain <domain>]
```

### 6.18 Configuration Management

```bash
# View current configuration (non-sensitive)
bgpx-cli config show

# Get specific config key
bgpx-cli config get [--key path.to.key]

# Set a live-reloadable configuration value
bgpx-cli config set --key sessions.max_sessions --value 5000

# Reload configuration from file
bgpx-cli reload

# Validate configuration file
bgpx-cli config check --file /etc/bgpx/node.toml
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

Structured output for scripting and automation:

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

### 7.3 Quiet Mode

Exit code only, no output (useful for scripting):

```bash
$ bgpx-cli --quiet paths build --segments "default:4"
$ echo $?
0  # Success
```

---

## 8. Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic error |
| 2 | Invalid arguments |
| 3 | Connection failed (daemon not running or socket unreachable) |
| 4 | Timeout |
| 5 | Permission denied |
| 6 | Not found |
| 7 | Daemon error |
| 8 | Configuration error |
| 9 | Operation failed (see stderr) |

---

## 9. Shell Completion

```bash
# Generate shell completion scripts
bgpx-cli completions bash > /etc/bash_completion.d/bgpx-cli
bgpx-cli completions zsh > ~/.zsh/completions/_bgpx-cli
bgpx-cli completions fish > ~/.config/fish/completions/bgpx-cli.fish
```

---

## 10. Standalone Mode vs. Router Mode

The control client connects to whichever daemon it can find. The connection model is the same:

**Standalone (daemon on same device)**: Connect to local control socket (`/var/run/bgpx/control.sock`)

**Router (managing from same router)**: SSH to router, use local control socket

**Router (managing remotely)**: SSH socket forwarding, use forwarded socket

The client has no "mode" — it just connects to whatever socket is configured.

---

## 11. Configuration Client vs SDK Application

| Aspect | Configuration Client | SDK Application |
|---|---|---|
| **Purpose** | Manage and monitor daemon | Build BGP-X-aware applications |
| **Connection** | Control Socket (`control.sock`) | SDK Socket (`sdk.sock`) |
| **Protocol** | JSON-RPC 2.0 (text only) | JSON-RPC 2.0 with binary extensions |
| **Capabilities** | Status, config, management | Open streams, register services, path control |
| **Typical user** | Operator / Administrator | Application developer |

The SDK is documented in `/sdk/sdk_spec.md`. The SDK connects to the SDK Socket (`/var/run/bgpx/sdk.sock`) and provides application-level functionality for building BGP-X-native applications.

---

## 12. GUI Alternatives

The Control API is JSON-RPC 2.0 over Unix socket. Any GUI tool that implements this interface is a valid control client.

### 12.1 LuCI Web Interface

Available via `bgpx-luci` OpenWrt package. Provides web-based management at `http://192.168.1.1/cgi-bin/luci/admin/bgpx/`.

Features:
- Dashboard with node status
- Routing policy editor
- Pool management
- Domain and island status
- Path monitoring
- Event log viewer

### 12.2 Planned Desktop GUI

A cross-platform desktop application (Electron-based) is planned. It will use the same JSON-RPC Control API as the CLI.

---

## 13. Security Model

### 13.1 Socket Permissions

The Control Socket should have restrictive permissions:

```
srwx------ 1 root root 0 Apr 24 12:00 /var/run/bgpx/control.sock
```

Only the `root` user or users in the `bgpx` group should be able to connect.

For multi-user access:
- Change group to `bgpx`
- Change permissions to `0660`

```bash
chown root:bgpx /var/run/bgpx/control.sock
chmod 660 /var/run/bgpx/control.sock
```

### 13.2 Authentication for TCP Mode

If TCP mode is enabled:

```toml
[control_api]
tcp_listen = "127.0.0.1:7476"
tcp_auth_required = true
tcp_auth_token_path = "/etc/bgpx/control_token"
```

Generate auth token:

```bash
# Generate token
openssl rand -hex 32 > /etc/bgpx/control_token
chmod 600 /etc/bgpx/control_token
```

Client usage:

```bash
bgpx-cli --tcp 127.0.0.1:7476 --token /etc/bgpx/control_token node show
```

### 13.3 No Logging of Sensitive Data

The client itself does not log sensitive information. All logging is performed by the daemon, which follows the strict no-log policy defined in the architecture.

**The daemon MUST NOT log**:
- Client IPs at relay/exit nodes
- Destinations at relay nodes
- path_id values
- Path composition
- Session IDs
- Traffic volume per path
- Pool query history
- Cross-domain traversal details per session
- Mesh island identifiers associated with specific sessions

---

## 14. Platform Support

| Platform | Support Level | Notes |
|---|---|---|
| Linux (x86_64, ARM) | Full | Primary target |
| macOS (x86_64, ARM) | Full | Unix socket works |
| Windows (x86_64) | Planned | Named pipe or TCP endpoint needed |
| Android | Planned | Via Termux or dedicated app |
| iOS | Not applicable | No terminal environment |

---

## 15. Example Usage Scenarios

### 15.1 Check Node Status

```bash
$ bgpx-cli node show
```

### 15.2 Build a Cross-Domain Path

```bash
$ bgpx-cli paths build example.com \
    --domain-segments '[
        {"type": "segment", "domain": "clearnet", "hops": 2},
        {"type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:lima-district-1"},
        {"type": "segment", "domain": "mesh:lima-district-1", "hops": 2}
    ]'
```

### 15.3 Register a BGP-X Native Service

```bash
$ bgpx-cli names register myservice \
    --service a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

### 15.4 Monitor Active Paths

```bash
$ bgpx-cli paths list --format json | jq '.[] | select(.quality_score < 0.5)'
```

### 15.5 Add a Routing Policy Rule

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

### 15.6 Monitor Mesh Island Connectivity

```bash
$ bgpx-cli islands connectivity-test lima-district-1 --from-clearnet
```

### 15.7 Test Domain Bridge Latency

```bash
$ bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1" --latency-report
```

---

## 16. Configuration File Location

The client itself does not use a mandatory configuration file. All connection parameters can be specified via:

### 16.1 Command-Line Arguments

```bash
bgpx-cli --socket /var/run/bgpx/control.sock node show
```

### 16.2 Environment Variables

```bash
export BGPX_CONTROL_SOCKET=/var/run/bgpx/control.sock
export BGPX_TIMEOUT=60
export BGPX_FORMAT=json

bgpx-cli node show   # Uses environment variables
```

### 16.3 Optional Configuration File

```toml
# ~/.config/bgpx/cli.toml

[connection]
socket_path = "/var/run/bgpx/control.sock"

[output]
format = "text"
quiet = false
verbose = false

[timeouts]
default = 30
```

---

## 17. Relationship to Other Components

| Document | Relationship |
|---|---|
| `/node/node.md` | Daemon that this client controls |
| `/node/api.md` | Internal API the daemon exposes |
| `/control-plane/control_api.md` | JSON-RPC API specification |
| `/sdk/sdk_spec.md` | SDK for building BGP-X-aware applications |

---

## 18. Summary

The BGP-X control client (`bgpx-cli`) is:

- A **management tool** for bgpx-node daemons
- A **JSON-RPC 2.0 client** connecting to the Control Socket
- A **CLI interface** for configuration, monitoring, and diagnostics
- **NOT a routing stack** — that is bgpx-node

For building BGP-X-aware applications, use the SDK (`/sdk/sdk_spec.md`).

For the daemon specification, see `/node/node.md`.
