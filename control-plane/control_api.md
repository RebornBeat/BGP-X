# BGP-X Control API Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X Control API — the interface by which local applications, management tools, and the BGP-X CLI interact with a running BGP-X node or client daemon.

---

## 1. Overview

The Control API is a local-only, Unix domain socket API. It is NOT exposed over the network and is NOT part of the BGP-X overlay protocol. It provides:

- Node status and statistics
- Path management
- Reputation database access
- Node advertisement management
- Pool management
- Routing policy management
- Domain and mesh island management
- ECH and session management
- Configuration updates
- Shutdown and lifecycle management

**Location**: `/var/run/bgpx/control.sock` (system) or `/tmp/bgpx-<uid>/control.sock` (user mode)

**Protocol**: JSON-RPC 2.0 over Unix domain socket, newline-delimited messages

**Authentication**: Unix socket file permissions. The socket file is owned by the user running the BGP-X daemon and has permissions 0600 (owner read/write only) by default.

**Network exposure**: MUST NOT be exposed over the network. Control socket is management-only, local-only. Network exposure of the Control API is not supported and MUST NOT be implemented.

Operators who wish to grant additional users API access should use group ownership and 0660 permissions rather than exposing the socket over the network.

---

## 2. Transport

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

## 3. Method Catalog

### 3.1 Node Lifecycle

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
  "advertisement_expires_at": "2026-04-25T12:00:00Z",
  "bridge_capable": true,
  "active_domains": ["clearnet", "mesh:lima-district-1"],
  "active_bridges": [
    {"from": "clearnet", "to": "mesh:lima-district-1", "available": true}
  ],
  "cross_domain_paths_active": 3,
  "mesh_active": true,
  "gateway_active": false,
  "pt_enabled": false,
  "ech_capable": true
}
```

State values: `INITIAL`, `BOOTSTRAP`, `ACTIVE`, `DRAIN`, `STOP`

Field descriptions:

| Field | Description |
|---|---|
| bridge_capable | Whether this node has endpoints in 2+ routing domains |
| active_domains | List of routing domains this node currently serves |
| active_bridges | Active bridge pairs with availability status |
| cross_domain_paths_active | Count of paths currently traversing multiple domains |
| mesh_active | Whether mesh transport is operational |
| gateway_active | Whether this node is operating as a mesh-to-clearnet gateway |
| pt_enabled | Whether pluggable transport is currently active |
| ech_capable | Whether this exit node supports ECH |

---

#### `shutdown`

Initiates graceful shutdown. The daemon enters DRAIN state, completes in-flight sessions, and exits.

**Parameters**:

```json
{
  "timeout_seconds": 30,
  "publish_withdrawal": true
}
```

If `publish_withdrawal = true` (default): daemon publishes NODE_WITHDRAW before draining. For bridge-capable nodes: also marks all bridge pairs as unavailable in DOMAIN_ADVERTISE and re-publishes.

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

### 3.2 Path Management

#### `list_paths`

Returns all currently active paths.

**Parameters**: none

**Result**:

```json
{
  "paths": [
    {
      "hops": 4,
      "age_seconds": 120,
      "active_streams": 3,
      "bytes_sent": 1048576,
      "bytes_received": 5242880,
      "latency_ms": 145,
      "quality_score": 0.82,
      "pool_per_segment": ["bgpx-default", "eu-no-log-exits"],
      "exit_jurisdiction": "CH",
      "exit_logging_policy": "none",
      "domain_segments": [
        {"domain": "clearnet", "hops": 3},
        {"domain": "clearnet", "hops": 1}
      ],
      "total_domains": 1,
      "is_cross_domain": false
    }
  ]
}
```

**Privacy**: path_id is NOT included in results. No node identifiers are exposed.

Field descriptions:

| Field | Description |
|---|---|
| domain_segments | Per-domain hop breakdown |
| total_domains | Number of routing domains traversed |
| is_cross_domain | Whether path crosses domain boundaries |

---

#### `build_path`

Requests the client to build a new path immediately.

**Parameters** (domain_segments format for cross-domain paths):

```json
{
  "domain_segments": [
    {"type": "segment", "domain": "clearnet", "pool": "bgpx-default", "hops": 2},
    {"type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:lima-district-1"},
    {"type": "segment", "domain": "mesh:lima-district-1", "hops": 2},
    {"type": "bridge", "from_domain": "mesh:lima-district-1", "to_domain": "clearnet"},
    {"type": "segment", "domain": "clearnet", "pool": "private-exits", "hops": 1, "exit": true}
  ],
  "constraints": {
    "exit_jurisdiction_blacklist": ["US", "GB"],
    "exit_logging_policy": "none",
    "require_ech": false,
    "min_reputation_score": 70
  }
}
```

**Parameters** (legacy segments format for single-domain paths):

```json
{
  "hops": 4,
  "segments": [
    {"pool": "bgpx-default", "hops": 3},
    {"pool": "my-private-exit", "hops": 1, "exit": true}
  ],
  "constraints": {
    "exit_jurisdiction_blacklist": ["US", "GB"],
    "exit_logging_policy": "none",
    "require_ech": false,
    "min_reputation_score": 70
  }
}
```

Both formats are accepted. The domain_segments format is required for cross-domain paths. The legacy segments format is retained for backward compatibility with single-domain paths.

**Result**:

```json
{
  "hops": 6,
  "build_time_ms": 342,
  "status": "ready",
  "domain_count": 3
}
```

**Note**: path_id is NOT returned to protect path composition privacy.

---

#### `destroy_path`

Destroys a specific path and tears down all its sessions.

**Parameters**:

```json
{
  "path_index": 0
}
```

`path_index` refers to the index in the list_paths result array.

**Result**:

```json
{
  "destroyed": true,
  "streams_closed": 3
}
```

---

### 3.3 Routing Policy Management

#### `list_rules`

List all routing policy rules in evaluation order.

**Parameters**: none

**Result**: ordered array of rules with IDs, match criteria, actions.

```json
{
  "rules": [
    {
      "rule_id": "rule-001",
      "priority": 10,
      "match": {
        "destination_domain": ["*.sensitive.com"],
        "destination_port": [443]
      },
      "action": "bgpx",
      "path_constraints": {
        "domain_segments": [
          {"type": "segment", "domain": "clearnet", "pool": "bgpx-default", "hops": 2},
          {"type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:trusted-island"},
          {"type": "segment", "domain": "mesh:trusted-island", "hops": 2},
          {"type": "bridge", "from_domain": "mesh:trusted-island", "to_domain": "clearnet"},
          {"type": "segment", "domain": "clearnet", "pool": "private-exits", "hops": 1, "exit": true}
        ],
        "require_ech": true
      },
      "reason": "high-security-via-mesh"
    }
  ]
}
```

---

#### `add_rule`

Add a new routing policy rule.

**Parameters**:

```json
{
  "position": 5,
  "match": {
    "destination_domain": ["*.sensitive.com"],
    "destination_port": [443]
  },
  "action": "bgpx",
  "path_constraints": {
    "hops": 6,
    "require_ech": true,
    "exit_logging_policy": "none"
  },
  "reason": "high-security-sites"
}
```

**Result**:

```json
{
  "rule_id": "rule-abc123",
  "position": 5
}
```

---

#### `remove_rule`

Remove a routing policy rule.

**Parameters**:

```json
{
  "rule_id": "rule-abc123"
}
```

**Result**:

```json
{
  "removed": true
}
```

---

#### `reorder_rule`

Change the priority position of a rule.

**Parameters**:

```json
{
  "rule_id": "rule-abc123",
  "new_position": 2
}
```

**Result**:

```json
{
  "reordered": true,
  "new_position": 2
}
```

---

#### `test_destination`

Show which rule and path would be used for a destination/device combination.

**Parameters**:

```json
{
  "destination": "example.com",
  "device_ip": "192.168.1.50",
  "protocol": "tcp",
  "port": 443
}
```

**Result**:

```json
{
  "matching_rule": {
    "rule_id": "rule-001",
    "action": "bgpx",
    "reason": "default-overlay"
  },
  "path_constraints": {
    "hops": 4,
    "pool": "bgpx-default"
  }
}
```

---

#### `reload_policy`

Reload routing policy from config file.

**Parameters**: none

**Result**:

```json
{
  "reloaded": true,
  "rules_loaded": 15
}
```

---

#### `set_routing_policy_default`

Set the default action for unmatched traffic.

**Parameters**:

```json
{
  "default_action": "bgpx"
}
```

Valid values: `bgpx`, `standard`, `block`

---

#### `get_routing_policy_stats`

Returns aggregate packet counts per rule category (no identifiers).

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
  "bgpx_count": 8940,
  "standard_count": 3210,
  "block_count": 15,
  "per_rule_counts": [
    {"rule_id": "rule-001", "count": 4200},
    {"rule_id": "rule-002", "count": 3100}
  ]
}
```

---

### 3.4 Pool Management

#### `list_pools`

List all known pools.

**Parameters**:

```json
{
  "filter": {
    "type": "curated",
    "min_active_members": 5,
    "domain_scope": "mesh:lima-district-1"
  }
}
```

**Result**:

```json
{
  "pools": [
    {
      "pool_id": "pool-id-hex",
      "name": "EU Curated Relays",
      "type": "curated",
      "member_count": 42,
      "active_member_count": 38,
      "regions": ["EU"],
      "domain_scope": null
    },
    {
      "pool_id": "pool-id-hex-2",
      "name": "Lima District Mesh Relays",
      "type": "public",
      "member_count": 8,
      "active_member_count": 7,
      "regions": ["SA"],
      "domain_scope": "mesh:lima-district-1"
    }
  ]
}
```

---

#### `get_pool`

Retrieve full pool details and member list.

**Parameters**:

```json
{
  "pool_id": "hex..."
}
```

**Result**: Full pool advertisement JSON including member list.

---

#### `add_pool`

Add a private pool (local config only, not published to DHT).

**Parameters**: Full pool advertisement JSON.

**Result**:

```json
{
  "added": true,
  "pool_id": "hex..."
}
```

---

#### `remove_pool`

Remove a private pool from local configuration.

**Parameters**:

```json
{
  "pool_id": "hex..."
}
```

**Result**:

```json
{
  "removed": true
}
```

---

#### `verify_pool`

Verify pool signature and check member availability.

**Parameters**:

```json
{
  "pool_id": "hex..."
}
```

**Result**:

```json
{
  "signature_valid": true,
  "member_count": 42,
  "active_count": 38,
  "issues": []
}
```

---

#### `refresh_pool`

Force re-fetch of pool member list from DHT.

**Parameters**:

```json
{
  "pool_id": "hex..."
}
```

**Result**:

```json
{
  "refreshed": true,
  "member_count": 42
}
```

---

#### `rotate_pool_key`

Trigger pool curator key rotation. (For nodes that are also pool curators.)

**Parameters**:

```json
{
  "pool_id": "hex...",
  "old_key_path": "/etc/bgpx/curator_key_old",
  "new_key_path": "/etc/bgpx/curator_key_new",
  "reason": "scheduled"
}
```

---

#### `list_mesh_pools`

List pools scoped to mesh island domains.

**Parameters**: none

**Result**: Array of pool summaries where `domain_scope` is a mesh island.

---

### 3.5 Node Database Management

#### `list_nodes`

Returns nodes from the local node database, with optional filtering.

**Parameters**:

```json
{
  "filter": {
    "roles": ["exit"],
    "region": "EU",
    "min_reputation": 70,
    "min_uptime_pct": 90,
    "pool": "pool-id-hex",
    "ech_capable": true,
    "mesh_transport": "lora",
    "domain": "mesh:lima-district-1",
    "bridge_capable": true,
    "bridge_to_domain": "clearnet"
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
      "blacklisted": false,
      "bridge_capable": false,
      "bridge_records": []
    }
  ]
}
```

---

#### `get_node`

Returns full details for a specific node including reputation event history (last 50 events).

**Parameters**:

```json
{
  "node_id": "hex..."
}
```

**Result**: Full node database record including:
- Complete advertisement fields
- Reputation score breakdown
- Last 50 reputation events
- Bridge records (if bridge_capable)
- Pool memberships
- Island memberships

---

#### `blacklist_node`

Manually blacklists a node. Overrides reputation-based evaluation.

**Parameters**:

```json
{
  "node_id": "hex...",
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

Removes a manual blacklist entry. Cannot remove permanent blacklist entries (cryptographic violations).

**Parameters**:

```json
{
  "node_id": "hex..."
}
```

**Result**:

```json
{
  "unblacklisted": true
}
```

---

#### `clear_blacklist_entry`

Remove expired blacklist entry for re-evaluation.

**Parameters**:

```json
{
  "node_id": "hex..."
}
```

**Result**:

```json
{
  "cleared": true
}
```

---

### 3.6 Advertisement Management (Node Daemon)

#### `get_advertisement`

Returns the node's current published advertisement.

**Parameters**: none

**Result**: Current signed node advertisement JSON, including `routing_domains` and `bridges` fields.

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

#### `withdraw_advertisement`

Publish NODE_WITHDRAW (for graceful shutdown from management tool).

**Parameters**:

```json
{
  "reason": "scheduled_maintenance"
}
```

**Result**:

```json
{
  "withdrawn": true,
  "propagated_to": 18
}
```

---

### 3.7 ECH Management (Exit Nodes)

#### `get_ech_cache`

Show cached ECH configurations.

**Parameters**: none

**Result**:

```json
{
  "cached_configs": [
    {
      "destination": "example.com",
      "ech_available": true,
      "config_source": "dns_https_record",
      "cached_at": "2026-04-24T12:00:00Z",
      "ttl_seconds": 3600
    }
  ]
}
```

---

#### `clear_ech_cache`

Force re-fetch of ECH configurations.

**Parameters**: none

**Result**:

```json
{
  "cleared": true,
  "entries_removed": 15
}
```

---

#### `test_ech`

Test ECH negotiation for a specific destination.

**Parameters**:

```json
{
  "destination": "example.com"
}
```

**Result**:

```json
{
  "ech_available": true,
  "ech_config_found": true,
  "negotiation_result": "success",
  "latency_ms": 145
}
```

---

### 3.8 SDK Socket Configuration

#### `get_sdk_socket_config`

Return current SDK socket configuration.

**Parameters**: none

**Result**:

```json
{
  "socket_path": "/var/run/bgpx/sdk.sock",
  "tcp_listen": "",
  "tcp_auth_required": false
}
```

---

#### `set_sdk_tcp_listen`

Enable TCP endpoint for LAN SDK access.

**Parameters**:

```json
{
  "listen_addr": "192.168.1.1:7475",
  "enabled": true
}
```

**Result**:

```json
{
  "updated": true,
  "listen_addr": "192.168.1.1:7475"
}
```

---

#### `set_sdk_tcp_auth`

Configure authentication for SDK TCP endpoint.

**Parameters**:

```json
{
  "auth_required": true,
  "token_path": "/etc/bgpx/sdk_auth_token"
}
```

**Result**:

```json
{
  "updated": true,
  "auth_required": true
}
```

---

### 3.9 Mesh Management

#### `list_transports`

Show active mesh transports and their status.

**Parameters**: none

**Result**:

```json
{
  "transports": [
    {
      "type": "wifi_mesh",
      "interface": "mesh0",
      "status": "active",
      "peers_connected": 4,
      "tx_bytes": 1048576,
      "rx_bytes": 524288
    },
    {
      "type": "lora",
      "device": "/dev/ttyUSB0",
      "status": "active",
      "frequency_mhz": 868.0,
      "spreading_factor": 7,
      "tx_packets": 142,
      "rx_packets": 89
    }
  ]
}
```

---

#### `get_transport_stats`

Per-transport statistics (packets, bytes, errors, RTT).

**Parameters**:

```json
{
  "transport_type": "lora"
}
```

**Result**: Detailed statistics for the specified transport.

---

#### `force_beacon`

Trigger immediate MESH_BEACON broadcast on all mesh transports.

**Parameters**: none

**Result**:

```json
{
  "beacon_sent": true,
  "transports": ["wifi_mesh", "lora"]
}
```

---

#### `get_mesh_peers`

List discovered mesh neighbors.

**Parameters**: none

**Result**:

```json
{
  "peers": [
    {
      "node_id": "a3f2...",
      "transport": "wifi_mesh",
      "last_seen": "2026-04-24T12:00:00Z",
      "rtt_ms": 12
    }
  ]
}
```

**Privacy**: No path or session identifiers exposed.

---

#### `get_gateway_status`

Gateway-specific status: mesh-to-internet bridge stats, DHT sync status.

**Parameters**: none

**Result**:

```json
{
  "is_gateway": true,
  "served_mesh_islands": ["lima-district-1"],
  "bridge_status": [
    {
      "from_domain": "clearnet",
      "to_domain": "mesh:lima-district-1",
      "available": true,
      "latency_ms": 25
    }
  ],
  "dht_sync_status": "active",
  "dht_records_propagated": 87
}
```

**Alias**: `get_domain_bridge_status` (retained for backward compatibility)

---

### 3.10 Session Management

#### `get_session_rehandshake_status`

Show sessions due for 24hr re-handshake.

**Parameters**: none

**Result**:

```json
{
  "sessions_requiring_rehandshake": 3,
  "oldest_session_age_hours": 26.5,
  "rehandshake_queue": [
    {"session_index": 0, "age_hours": 26.5},
    {"session_index": 1, "age_hours": 25.2}
  ]
}
```

---

#### `force_session_rehandshake`

Manually trigger re-handshake for all sessions exceeding age threshold.

**Parameters**:

```json
{
  "min_age_hours": 24
}
```

**Result**:

```json
{
  "rehandshakes_initiated": 3,
  "rehandshakes_completed": 2,
  "rehandshakes_failed": 1
}
```

---

### 3.11 Statistics

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
  "path_constructions_failed": 1,
  "cross_domain_path_constructions": 12,
  "domain_bridge_transitions_total": 34,
  "mesh_island_paths_active": 3,
  "pool_count": 3,
  "pool_construction_attempts": 12,
  "geo_plausibility_events": 5,
  "ech_connections_total": 89,
  "ech_fallback_total": 3,
  "mesh_sessions_active": 4,
  "pt_active": false,
  "routing_policy_bgpx_count": 8940,
  "routing_policy_standard_count": 3210,
  "routing_policy_block_count": 15,
  "cross_domain_path_table_size": 18,
  "bridge_nodes_in_database": 7,
  "island_advertisements_cached": 3
}
```

**Privacy**: No session identifiers, node identifiers (beyond counts), or client IP addresses are included in statistics.

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

### 3.12 Domain Management

#### `list_domains`

List all routing domains known to this node.

**Parameters**: none

**Result**:

```json
{
  "domains": [
    {
      "domain_id": "0x00000001-00000000",
      "domain_type": "clearnet",
      "served_by_this_node": true,
      "active_relay_count": 0,
      "bridge_node_count": 15
    },
    {
      "domain_id": "0x00000003-a1b2c3d4",
      "domain_type": "mesh",
      "island_id": "lima-district-1",
      "served_by_this_node": true,
      "active_relay_count": 8,
      "bridge_node_count": 2,
      "online": true,
      "last_seen": "2026-04-24T12:00:00Z"
    }
  ]
}
```

---

#### `list_domain_bridges`

List domain bridge nodes for a specific domain pair.

**Parameters**:

```json
{
  "from_domain": "clearnet",
  "to_domain": "mesh:lima-district-1",
  "min_reputation": 70.0
}
```

**Result**:

```json
{
  "bridges": [
    {
      "from_domain": "clearnet",
      "to_domain": "mesh:lima-district-1",
      "available_count": 2,
      "best_latency_ms": 25,
      "average_reputation": 85.3
    }
  ]
}
```

**Privacy**: No individual node IDs in result — reputation and availability aggregates only.

---

#### `get_island_status`

Get status of a specific mesh island.

**Parameters**:

```json
{
  "island_id": "lima-district-1"
}
```

**Result**:

```json
{
  "island_id": "lima-district-1",
  "domain_id": "0x00000003-a1b2c3d4",
  "online": true,
  "active_relay_count": 8,
  "bridge_node_count": 2,
  "registered_services": 1,
  "dht_coverage": "full",
  "last_advertisement": "2026-04-24T12:00:00Z"
}
```

---

#### `list_islands`

List all known mesh islands.

**Parameters**:

```json
{
  "filter": {
    "online_only": true,
    "has_services": true
  }
}
```

**Result**: Array of island summaries.

---

#### `test_domain_connectivity`

Test reachability to a target domain.

**Parameters**:

```json
{
  "target_domain": "mesh:lima-district-1",
  "via_bridge": null
}
```

**Result**:

```json
{
  "reachable": true,
  "bridge_node_count": 2,
  "best_bridge_latency_ms": 25,
  "test_path_constructed": true,
  "test_path_hops": 6
}
```

---

#### `discover_bridge_nodes`

Force-fetch bridge node records from DHT for a domain pair.

**Parameters**:

```json
{
  "from_domain": "clearnet",
  "to_domain": "mesh:lima-district-1"
}
```

**Result**:

```json
{
  "discovered": 2,
  "dht_nodes_queried": 18
}
```

---

### 3.13 Configuration

#### `get_config`

Returns current active configuration (non-sensitive fields only — key paths are shown but key material is never returned).

**Parameters**: none

**Result**: Current configuration as JSON, with sensitive fields redacted. Includes `routing_domains` and `domain_bridge` configuration sections.

---

#### `set_config`

Update specific configuration values that support live reload.

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

## 4. Error Codes

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
| -32007 | Pool not found |
| -32008 | Pool signature invalid |
| -32009 | Pool insufficient nodes |
| -32010 | SDK TCP endpoint requires auth token |
| -32011 | ECH test failed |
| -32012 | Mesh transport not available |
| -32013 | Domain not found |
| -32014 | Domain bridge unavailable |
| -32015 | Mesh island unreachable |
| -32016 | Cross-domain path construction failed |
| -32017 | Domain scope mismatch in pool |
| -32018 | No cross-domain path found |

---

## 5. Event Subscriptions

Subscribe to a stream of events for monitoring applications.

#### `subscribe`

Subscribe to a stream of events.

**Parameters**:

```json
{
  "events": [
    "session_opened",
    "session_closed",
    "path_built",
    "path_degraded",
    "path_rebuilt",
    "node_blacklisted",
    "keepalive_timeout",
    "pool_member_offline",
    "pool_refresh_failed",
    "pool_key_rotated",
    "geo_plausibility_alert",
    "node_withdrawn",
    "mesh_beacon_received",
    "gateway_bridge_change",
    "session_rehandshake_triggered",
    "daemon_state_changed",
    "domain_bridge_online",
    "domain_bridge_offline",
    "mesh_island_discovered",
    "mesh_island_unreachable",
    "mesh_island_online",
    "cross_domain_path_built",
    "cross_domain_path_rebuilt",
    "bridge_availability_changed",
    "blacklist_entry_expired"
  ]
}
```

After subscribing, the daemon sends newline-delimited JSON event objects on the same socket connection:

```json
{"event": "session_opened", "timestamp": "2026-04-24T12:00:01Z", "session_count": 43}
{"event": "keepalive_timeout", "timestamp": "2026-04-24T12:00:45Z"}
{"event": "path_rebuilt", "timestamp": "2026-04-24T12:01:00Z", "hops": 4, "reason": "quality_degraded"}
{"event": "node_blacklisted", "timestamp": "2026-04-24T12:02:00Z"}
{"event": "pool_member_offline", "timestamp": "2026-04-24T12:03:00Z", "pool_id": "pool-hex..."}
{"event": "domain_bridge_online", "timestamp": "2026-04-24T12:04:00Z", "from_domain": "clearnet", "to_domain": "mesh:lima-district-1"}
{"event": "mesh_island_discovered", "timestamp": "2026-04-24T12:05:00Z", "island_id": "bogota-district-2"}
{"event": "cross_domain_path_built", "timestamp": "2026-04-24T12:06:00Z", "domain_count": 3}
```

**Privacy**: Event data MUST NOT include node IDs that could be linked to paths, session IDs, client IPs, or path composition.

---

## 6. CLI Usage

The BGP-X CLI (`bgpx-cli`) is a thin wrapper over the Control API. It connects to the local control socket and formats responses for human consumption.

Example CLI commands:

```bash
# Node lifecycle
bgpx-cli status
bgpx-cli shutdown --timeout 30
bgpx-cli reload-config

# Path management
bgpx-cli paths list
bgpx-cli paths build --hops 5 --exit-country DE --no-log
bgpx-cli paths build \
    --domain-segments "clearnet:2,bridge:clearnet→mesh:lima-1,mesh:lima-1:2"
bgpx-cli paths destroy --index 0

# Routing policy
bgpx-cli policy list
bgpx-cli policy add --destination sensitive.com --action bgpx --hops 6 --require-ech
bgpx-cli policy test --destination example.com --device 192.168.1.50
bgpx-cli policy reload

# Pools
bgpx-cli pools list
bgpx-cli pools verify --pool-id hex...
bgpx-cli pools refresh --pool-id hex...

# Node management
bgpx-cli nodes list --role exit --region EU --ech-capable
bgpx-cli nodes list --domain "mesh:lima-district-1" --bridge-capable
bgpx-cli nodes list --bridge-to "clearnet"
bgpx-cli nodes blacklist <node_id> --reason "suspected MITM"
bgpx-cli nodes unblacklist <node_id>

# Domain management
bgpx-cli domains list
bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1"
bgpx-cli domains island-status --island-id lima-district-1
bgpx-cli domains list-islands
bgpx-cli domains test --from clearnet --to "mesh:lima-district-1"
bgpx-cli domains discover-bridges --from clearnet --to "mesh:lima-district-1"

# Statistics
bgpx-cli stats --window 1h

# Mesh
bgpx-cli mesh peers
bgpx-cli mesh beacon
bgpx-cli mesh gateway-status

# ECH (exit nodes)
bgpx-cli exit ech-cache
bgpx-cli exit test-ech --destination example.com

# Session management
bgpx-cli sessions rehandshake-status
bgpx-cli sessions force-rehandshake

# Events
bgpx-cli events watch
bgpx-cli events watch --filter domain_bridge,mesh_island,cross_domain_path
```

---

## 7. Authentication Details

Access to the Control API is controlled by Unix socket file permissions. The socket file is owned by the user running the BGP-X daemon and has permissions 0600 (owner read/write only) by default.

There is no token-based authentication for the local Control API. Network exposure of the Control API is not supported and MUST NOT be implemented.

For multi-user access on a shared system:

1. Create a `bgpx` group
2. Set socket permissions to 0660
3. Add authorized users to the `bgpx` group

Example configuration in daemon config:

```toml
[control_api]
socket_path = "/var/run/bgpx/control.sock"
socket_group = "bgpx"
socket_mode = "0660"
```

---

## 8. Security Considerations

- The Control API must never be exposed to the network
- All methods that could affect system configuration require appropriate permissions
- No sensitive key material is ever returned via the API
- Event subscriptions do not include path or session identifiers
- Statistics contain no personally identifying information
