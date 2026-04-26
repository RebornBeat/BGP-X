# BGP-X Control API Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The Control API is a local-only Unix domain socket API. It provides full daemon management capabilities. It is NOT exposed over the network and is NOT part of the BGP-X overlay protocol.

**Location**: `/var/run/bgpx/control.sock` (system) or `/tmp/bgpx-<uid>/control.sock` (user mode)

**Protocol**: JSON-RPC 2.0 over Unix domain socket, newline-delimited messages

**Authentication**: Unix socket file permissions (0600 by default)

**Network exposure**: MUST NOT be exposed over the network.

---

## 2. Transport

```json
// Request
{"jsonrpc": "2.0", "id": 1, "method": "get_status", "params": {}}

// Response
{"jsonrpc": "2.0", "id": 1, "result": { ... }}

// Error
{"jsonrpc": "2.0", "id": 1, "error": {"code": -32601, "message": "Method not found"}}
```

---

## 3. Method Catalog

### 3.1 Node Lifecycle

#### `get_status`

```json
{
  "state": "ACTIVE",
  "uptime_seconds": 86400,
  "version": "0.1.0",
  "node_id": "a3f2...",
  "roles": ["relay", "entry"],
  "active_sessions": 42,
  "active_streams": 117,
  "bytes_relayed_total": 10737418240,
  "advertisement_expires_at": "2026-04-25T12:00:00Z",
  "bridge_capable": true,
  "active_domains": ["clearnet", "mesh:lima-district-1"],
  "active_bridges": [{"from": "clearnet", "to": "mesh:lima-district-1", "available": true}],
  "cross_domain_paths_active": 3,
  "mesh_active": true,
  "gateway_active": true,
  "pt_enabled": false,
  "ech_capable": true
}
```

#### `shutdown`

Parameters: `{"timeout_seconds": 30, "publish_withdrawal": true}`

#### `reload_config`

Reload config from disk. Returns reloaded and requires-restart options.

---

### 3.2 Path Management

#### `list_paths`

Returns all active paths. Per-path: hops, age, active_streams, bytes_sent/received, latency_ms, quality_score, pool_per_segment, exit_jurisdiction, exit_logging_policy, domain_segments, total_domains, is_cross_domain. No path_id or node identifiers.

#### `build_path`

```json
{
  "domain_segments": [
    { "type": "segment", "domain": "clearnet", "pool": "bgpx-default", "hops": 2 },
    { "type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:lima-district-1" },
    { "type": "segment", "domain": "mesh:lima-district-1", "hops": 2 }
  ],
  "constraints": {
    "exit_jurisdiction_blacklist": ["US"],
    "exit_logging_policy": "none",
    "require_ech": false,
    "min_reputation_score": 70
  }
}
```

Legacy `segments` (pool-only) format also accepted for single-domain paths.

#### `destroy_path`

Parameters: `{"path_index": 0}` (index in list_paths result).

---

### 3.3 Routing Policy Management

#### `list_rules`

List all routing policy rules in evaluation order with IDs, match criteria, actions.

#### `add_rule`

```json
{
  "position": 5,
  "match": {
    "destination_domain": ["*.sensitive.com"]
  },
  "action": "bgpx",
  "path_constraints": {
    "domain_segments": [
      { "type": "segment", "domain": "clearnet", "pool": "bgpx-default", "hops": 2 },
      { "type": "bridge", "from_domain": "clearnet", "to_domain": "mesh:trusted-island" },
      { "type": "segment", "domain": "mesh:trusted-island", "hops": 2 },
      { "type": "bridge", "from_domain": "mesh:trusted-island", "to_domain": "clearnet" },
      { "type": "segment", "domain": "clearnet", "pool": "private-exits", "hops": 1, "exit": true }
    ],
    "require_ech": true
  },
  "reason": "high-security-via-mesh"
}
```

#### `remove_rule`

Parameters: `{"rule_id": "rule-abc123"}`

#### `reorder_rule`

Parameters: `{"rule_id": "rule-abc123", "new_position": 2}`

#### `test_destination`

Parameters: `{"destination": "example.com", "device_ip": "192.168.1.50", "protocol": "tcp", "port": 443}`

Returns: matching rule, action, path constraints that would be applied (including any domain_segments).

#### `reload_policy`

#### `set_routing_policy_default`

Parameters: `{"default_action": "bgpx"}` or `{"default_action": "standard"}`

#### `get_routing_policy_stats`

Returns aggregate packet counts per rule category. No identifiers.

---

### 3.4 Pool Management

#### `list_pools`

Parameters: `{"filter": {"type": "curated", "min_active_members": 5, "domain_scope": "mesh:lima-district-1"}}`

Result includes `domain_scope` field per pool.

#### `get_pool`

Retrieve full pool details and member list.

#### `add_pool`

Add a private pool. Accepts domain-scoped pool advertisements.

#### `remove_pool`, `verify_pool`, `refresh_pool`, `rotate_pool_key`

All unchanged.

#### `list_mesh_pools`

List pools scoped to mesh island domains.

---

### 3.5 Node Database Management

#### `list_nodes`

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

#### `get_node`

Full details including reputation event history (last 50 events) and bridge records.

#### `blacklist_node`, `unblacklist_node`, `clear_blacklist_entry`

All unchanged.

---

### 3.6 Advertisement Management

#### `get_advertisement`, `republish_advertisement`, `update_performance_metrics`, `withdraw_advertisement`

All unchanged. `get_advertisement` result now includes `routing_domains` and `bridges` fields.

---

### 3.7 ECH Management

#### `get_ech_cache`, `clear_ech_cache`, `test_ech`

All unchanged.

---

### 3.8 SDK Socket Configuration

#### `get_sdk_socket_config`, `set_sdk_tcp_listen`, `set_sdk_tcp_auth`

All unchanged.

---

### 3.9 Mesh Management

#### `list_transports`, `get_transport_stats`, `force_beacon`, `get_mesh_peers`, `get_gateway_status`

All unchanged. `get_gateway_status` renamed to `get_domain_bridge_status` (alias retained for backward compatibility).

---

### 3.10 Session Management

#### `get_session_rehandshake_status`, `force_session_rehandshake`

All unchanged.

---

### 3.11 Statistics

#### `get_stats`

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

No session identifiers, node identifiers beyond counts, or client IP addresses.

---

### 3.12 Domain Management (New Methods)

#### `list_domains`

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

#### `list_domain_bridges`

Parameters: `{"from_domain": "clearnet", "to_domain": "mesh:lima-district-1", "min_reputation": 70.0}`

Result: list of bridge node summaries (no node IDs in exported result — reputation and availability aggregates only).

#### `get_island_status`

Parameters: `{"island_id": "lima-district-1"}`

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

#### `list_islands`

Parameters: `{"filter": {"online_only": true, "has_services": true}}`

#### `test_domain_connectivity`

Parameters: `{"target_domain": "mesh:lima-district-1", "via_bridge": null}`

```json
{
  "reachable": true,
  "bridge_node_count": 2,
  "best_bridge_latency_ms": 25,
  "test_path_constructed": true,
  "test_path_hops": 6
}
```

#### `discover_bridge_nodes`

Force-fetch bridge node records from DHT for a domain pair.

Parameters: `{"from_domain": "clearnet", "to_domain": "mesh:lima-district-1"}`

---

### 3.13 Configuration

#### `get_config`, `set_config`

`get_config` result now includes `routing_domains` and `domain_bridge` configuration sections (key paths shown, key material never returned).

---

## 4. Error Codes

All previous error codes retained. New:

| Code | Meaning |
|---|---|
| -32013 | Domain not found |
| -32014 | Domain bridge unavailable |
| -32015 | Mesh island unreachable |
| -32016 | Cross-domain path construction failed |
| -32017 | Domain scope mismatch in pool |
| -32018 | No cross-domain path found |

---

## 5. Event Subscriptions

All previous events retained. New events:

| Event | Description |
|---|---|
| `domain_bridge_online` | A domain bridge node for a watched pair became active |
| `domain_bridge_offline` | A domain bridge node became unreachable |
| `mesh_island_discovered` | A new mesh island advertisement found in DHT |
| `mesh_island_unreachable` | All bridges to a mesh island are offline |
| `mesh_island_online` | Mesh island regained bridge connectivity |
| `cross_domain_path_built` | A cross-domain path was successfully constructed |
| `cross_domain_path_rebuilt` | A cross-domain path was rebuilt after failure |
| `bridge_availability_changed` | A bridge pair's availability status changed |

Event data MUST NOT include node IDs linked to paths, session IDs, client IPs, or path composition.

---

## 6. CLI Usage Examples

All previous CLI commands retained. New:

```bash
# Domain management
bgpx-cli domains list
bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1"
bgpx-cli domains island-status --island-id lima-district-1
bgpx-cli domains list-islands
bgpx-cli domains test --from clearnet --to "mesh:lima-district-1"
bgpx-cli domains discover-bridges --from clearnet --to "mesh:lima-district-1"

# Cross-domain path building
bgpx-cli paths build \
    --domain-segments "clearnet:2,bridge:clearnet→mesh:lima-1,mesh:lima-1:2"

# Standard single-domain path building (unchanged)
bgpx-cli paths build --segments "default:3,private-exits:1"

# Extended node filtering
bgpx-cli nodes list --domain "mesh:lima-district-1" --bridge-capable
bgpx-cli nodes list --bridge-to "clearnet"

# Events including cross-domain
bgpx-cli events watch --filter domain_bridge,mesh_island,cross_domain_path
```
