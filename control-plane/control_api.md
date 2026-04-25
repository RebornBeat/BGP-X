# BGP-X Control API Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X Control API — the interface by which local applications, management tools, and the BGP-X CLI interact with a running BGP-X node or client daemon.

---

## 1. Overview

The Control API is a local-only, Unix domain socket API. It is not exposed over the network and is not part of the BGP-X overlay protocol. It provides:

- Node status and statistics
- Path management
- Reputation database access
- Node advertisement management
- Configuration updates
- Shutdown and lifecycle management

The Control API uses JSON-RPC 2.0 over a Unix domain socket at:

```
/var/run/bgpx/control.sock  (default, root-owned)
/tmp/bgpx-<uid>/control.sock  (user mode)
```

---

## 2. Authentication

Access to the Control API is controlled by Unix socket file permissions. The socket file is owned by the user running the BGP-X daemon and has permissions 0600 (owner read/write only) by default.

Operators who wish to grant additional users API access should use group ownership and 0660 permissions rather than exposing the socket over the network.

There is no token-based authentication for the local Control API. Network exposure of the Control API is not supported and MUST NOT be implemented.

---

## 3. JSON-RPC 2.0 Transport

Requests and responses are newline-delimited JSON-RPC 2.0 messages over the Unix domain socket.

Example request:

```json
{"jsonrpc": "2.0", "id": 1, "method": "get_status", "params": {}}
```

Example response:

```json
{"jsonrpc": "2.0", "id": 1, "result": {"state": "ACTIVE", "uptime_seconds": 3600}}
```

Example error response:

```json
{"jsonrpc": "2.0", "id": 1, "error": {"code": -32601, "message": "Method not found"}}
```

---

## 4. Method Catalog

### 4.1 Node lifecycle methods

#### `get_status`

Returns the current operational state of the node or client daemon.

**Parameters**: none

**Result**:

```json
{
  "state": "ACTIVE",
  "uptime_seconds": 86400,
  "version": "0.1.0",
  "node_id": "a3f2...b9c1",
  "roles": ["relay", "entry"],
  "active_sessions": 42,
  "active_streams": 117,
  "bytes_relayed_total": 10737418240,
  "advertisement_expires_at": "2026-04-25T12:00:00Z"
}
```

State values: `INITIAL`, `BOOTSTRAP`, `ACTIVE`, `DRAIN`, `STOP`

---

#### `shutdown`

Initiates graceful shutdown. The daemon enters DRAIN state, completes in-flight sessions, and exits.

**Parameters**:

```json
{
  "timeout_seconds": 30
}
```

**Result**:

```json
{
  "accepted": true,
  "drain_timeout_seconds": 30
}
```

---

#### `reload_config`

Reloads the node configuration from disk without restarting. Not all configuration options can be reloaded without restart — non-reloadable options are listed in the result.

**Parameters**: none

**Result**:

```json
{
  "reloaded": true,
  "reloaded_options": ["max_sessions", "bandwidth_limit_mbps"],
  "requires_restart": ["listen_port", "node_private_key_path"]
}
```

---

### 4.2 Path management methods (client daemon only)

#### `list_paths`

Returns all currently active paths.

**Parameters**: none

**Result**:

```json
{
  "paths": [
    {
      "path_id": "550e8400-e29b-41d4-a716-446655440000",
      "hops": 4,
      "age_seconds": 120,
      "active_streams": 3,
      "bytes_sent": 1048576,
      "bytes_received": 5242880,
      "latency_ms": 145,
      "status": "healthy"
    }
  ]
}
```

---

#### `build_path`

Requests the client to build a new path immediately.

**Parameters**:

```json
{
  "hops": 4,
  "constraints": {
    "exit_jurisdiction_blacklist": ["US", "GB"],
    "exit_logging_policy": "none",
    "min_reputation_score": 70
  }
}
```

**Result**:

```json
{
  "path_id": "550e8400-e29b-41d4-a716-446655440001",
  "hops": 4,
  "build_time_ms": 342,
  "status": "ready"
}
```

---

#### `destroy_path`

Destroys a specific path and tears down all its sessions.

**Parameters**:

```json
{
  "path_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Result**:

```json
{
  "destroyed": true,
  "streams_closed": 3
}
```

---

### 4.3 Node database methods

#### `list_nodes`

Returns nodes from the local node database, with optional filtering.

**Parameters**:

```json
{
  "filter": {
    "roles": ["exit"],
    "region": "EU",
    "min_reputation": 70,
    "min_uptime_pct": 90
  },
  "limit": 50,
  "offset": 0,
  "sort_by": "reputation_score",
  "sort_order": "desc"
}
```

**Result**:

```json
{
  "total": 87,
  "nodes": [
    {
      "node_id": "a3f2...b9c1",
      "roles": ["relay", "entry"],
      "country": "DE",
      "region": "EU",
      "asn": 12345,
      "reputation_score": 91.2,
      "uptime_pct": 99.1,
      "latency_ms": 12,
      "bandwidth_mbps": 100,
      "advertisement_expires_at": "2026-04-25T12:00:00Z",
      "blacklisted": false
    }
  ]
}
```

---

#### `get_node`

Returns full details for a specific node.

**Parameters**:

```json
{
  "node_id": "a3f2...b9c1"
}
```

**Result**: Full node database record including reputation event history (last 50 events).

---

#### `blacklist_node`

Manually blacklists a node. Overrides reputation-based evaluation.

**Parameters**:

```json
{
  "node_id": "a3f2...b9c1",
  "reason": "manual_operator_blacklist",
  "note": "Operator reported this node is acting maliciously",
  "duration_hours": 720
}
```

**Result**:

```json
{
  "blacklisted": true,
  "effective_until": "2026-05-24T12:00:00Z"
}
```

---

#### `unblacklist_node`

Removes a manual blacklist entry. Cannot remove permanent blacklist entries (cryptographic violation).

**Parameters**:

```json
{
  "node_id": "a3f2...b9c1"
}
```

**Result**:

```json
{
  "unblacklisted": true
}
```

---

### 4.4 Advertisement management methods (node daemon only)

#### `get_advertisement`

Returns the node's current published advertisement.

**Parameters**: none

**Result**: Current signed node advertisement JSON.

---

#### `republish_advertisement`

Forces immediate re-publication of the node advertisement to the DHT.

**Parameters**: none

**Result**:

```json
{
  "published": true,
  "storage_nodes_reached": 18,
  "new_expires_at": "2026-04-25T12:00:00Z"
}
```

---

#### `update_performance_metrics`

Updates the self-reported performance metrics in the node advertisement.

**Parameters**:

```json
{
  "bandwidth_mbps": 200,
  "latency_ms": 10,
  "uptime_pct": 99.5
}
```

**Result**:

```json
{
  "updated": true,
  "republished": true
}
```

---

### 4.5 Statistics methods

#### `get_stats`

Returns aggregate operational statistics.

**Parameters**:

```json
{
  "window_seconds": 3600
}
```

**Result**:

```json
{
  "window_seconds": 3600,
  "packets_relayed": 142857,
  "bytes_relayed": 2147483648,
  "sessions_opened": 423,
  "sessions_closed": 411,
  "handshake_successes": 423,
  "handshake_failures": 2,
  "keepalive_timeouts": 5,
  "decryption_failures": 0,
  "replay_attempts": 1,
  "exit_policy_denials": 12,
  "dht_queries_sent": 890,
  "dht_queries_received": 2341,
  "node_advertisements_stored": 87,
  "reputation_positive_events": 1420,
  "reputation_negative_events": 8,
  "blacklisted_nodes": 2,
  "path_constructions_attempted": 45,
  "path_constructions_succeeded": 44,
  "path_constructions_failed": 1
}
```

Note: No session identifiers, node identifiers (beyond counts), or client IP addresses are included in statistics.

---

#### `get_dht_stats`

Returns DHT-specific statistics.

**Parameters**: none

**Result**:

```json
{
  "routing_table_size": 342,
  "buckets_active": 28,
  "estimated_network_size": 4200,
  "bootstrap_nodes_reachable": 5,
  "last_advertisement_publish": "2026-04-24T06:00:00Z",
  "storage_records_held": 87
}
```

---

### 4.6 Configuration methods

#### `get_config`

Returns the current active configuration (non-sensitive fields only — key paths are shown but key material is never returned).

**Parameters**: none

**Result**: Current configuration as JSON, with sensitive fields redacted.

---

#### `set_config`

Updates specific configuration values that support live reload.

**Parameters**:

```json
{
  "max_sessions": 1000,
  "bandwidth_limit_mbps": 500,
  "keepalive_interval_seconds": 25
}
```

**Result**:

```json
{
  "updated": ["max_sessions", "bandwidth_limit_mbps", "keepalive_interval_seconds"],
  "requires_restart": []
}
```

---

## 5. Error Codes

BGP-X Control API uses standard JSON-RPC 2.0 error codes plus custom codes:

| Code | Meaning |
|---|---|
| -32700 | Parse error |
| -32600 | Invalid request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 | Node not found |
| -32001 | Path not found |
| -32002 | Daemon not in correct state for this operation |
| -32003 | Permission denied |
| -32004 | Path construction failed |
| -32005 | Cannot unblacklist permanent blacklist |
| -32006 | Configuration value not live-reloadable |

---

## 6. Streaming Events (Subscriptions)

The Control API supports event subscriptions for monitoring applications.

#### `subscribe`

Subscribe to a stream of events.

**Parameters**:

```json
{
  "events": ["session_opened", "session_closed", "path_built", "node_blacklisted", "keepalive_timeout"]
}
```

After subscribing, the daemon sends newline-delimited JSON event objects on the same socket connection:

```json
{"event": "session_opened", "timestamp": "2026-04-24T12:00:01Z", "session_count": 43}
{"event": "keepalive_timeout", "timestamp": "2026-04-24T12:00:45Z"}
{"event": "path_built", "timestamp": "2026-04-24T12:01:00Z", "hops": 4, "build_time_ms": 280}
```

Note: Event data MUST NOT include node IDs, session IDs, or any information that could identify specific nodes or clients.

---

## 7. CLI Usage

The BGP-X CLI (`bgpx-cli`) is a thin wrapper over the Control API. It connects to the local control socket and formats responses for human consumption.

Example CLI commands:

```bash
# Check node status
bgpx-cli status

# List active paths
bgpx-cli paths list

# Build a new path with constraints
bgpx-cli paths build --hops 5 --exit-country DE --no-log

# List nodes by reputation
bgpx-cli nodes list --role exit --region EU --min-reputation 80

# Blacklist a node
bgpx-cli nodes blacklist <node_id> --reason "suspected MITM" --duration 720h

# View statistics
bgpx-cli stats --window 1h

# Watch live events
bgpx-cli events watch
```
