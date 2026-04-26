# BGP-X DHT Pool Specification

**Version**: 0.1.0-draft

---

## 1. Overview

A DHT Pool is a named, signed collection of BGP-X nodes grouped by trust level, operational purpose, or routing domain. Pools enable path construction that routes different segments through different trusted node groups.

Pool membership is a reputation signal and selection mechanism — NOT a bypass for individual node verification. Each pool member's individual advertisement is verified independently.

---

## 2. Pool Types

| Type | Description | Membership Discovery | Who Can Join |
|---|---|---|---|
| Public | Open collection of nodes that opt in | DHT lookup | Any node |
| Curated | Curator vets and signs membership | DHT lookup | Curator-approved nodes |
| Private | Membership shared out-of-band | Out-of-band | By invitation |
| Semi-private | Pool ID discoverable, join requires approval | DHT lookup | By approval |
| Ephemeral | Session-scoped, not published | Direct communication | Client-specified |
| Domain-scoped | Restricted to nodes in a specific routing domain | DHT lookup with domain filter | Domain-eligible nodes |
| Domain-bridge | Composed of bridge nodes for a specific domain pair | DHT lookup | Bridge-capable nodes only |

---

## 3. Pool Advertisement JSON

```json
{
  "version": 1,
  "pool_id": "<BLAKE3 hash, hex, 32 bytes>",
  "pool_name": "EU No-Log Exit Nodes",
  "pool_type": "curated",
  "curator_public_key": "<Ed25519 public key, base64url>",
  "member_list_format": "embedded",
  "members": ["node_id_1_hex", "node_id_2_hex"],
  "member_count": 15,
  "description": "Exit nodes in EU with verified no-log policies",
  "domain_scope": "any",
  "bridges": null,
  "min_reputation_for_selection": 70.0,
  "regions_covered": ["EU"],
  "min_uptime_pct": 95.0,
  "exit_policies_required": true,
  "exit_logging_policy_required": "none",
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-05-01T12:00:00Z",
  "signature": "<Ed25519 signature over canonical JSON>"
}
```

### Field: `domain_scope`

Restricts pool membership to nodes in a specific routing domain.

Values:
- `"any"` (default): pool contains nodes from any routing domain — backward compatible with all existing pool usage
- `"clearnet"`: pool contains only clearnet-reachable nodes
- `"mesh:<island_id>"`: pool contains only nodes in the specified mesh island
- `"lora-regional:<region_id>"`: pool contains only nodes in the specified LoRa region

### Field: `bridges` (for domain-bridge pools)

When `pool_type = "domain_bridge"`, describes the specific bridge pair this pool serves:

```json
{
  "pool_type": "domain_bridge",
  "bridges": {
    "from_domain": "0x00000001-00000000",
    "to_domain": "0x00000003-a1b2c3d4",
    "from_domain_name": "clearnet",
    "to_domain_name": "mesh:lima-district-1"
  }
}
```

A domain-bridge pool contains only nodes that bridge the specified domain pair. Each member's individual advertisement MUST confirm it serves both domains. Pool membership alone is insufficient — individual node advertisement verification REQUIRED.

### Field: `member_list_format`

For large pools (>100 members), `member_list_format = "external"` with a `member_endpoint` URL:
```json
{
  "member_list_format": "external",
  "member_endpoint": "https://pool-registry.example.com/pools/pool-id/members",
  "member_endpoint_signature_key": "<Ed25519 public key, base64url>"
}
```

The external member list response is separately signed.

---

## 4. Pool Discovery Protocol

### DHT Storage Key

```
pool_key = BLAKE3("bgpx-pool-v1" || pool_id)
TTL: 7 days. Re-publication: every 3 days.
```

### Domain-Scoped Pool Discovery

Domain-scoped pool discovery adds a verification step after standard retrieval:

```
1. Retrieve pool advertisement via DHT_GET(pool_key)
2. Verify pool advertisement signature
3. Verify pool.domain_scope matches required path segment domain
   If mismatch: ERR_DOMAIN_SCOPE_MISMATCH
4. Cache pool advertisement
5. For each listed member: fetch and verify individual node advertisement
6. Verify each member serves the pool's domain_scope domain
```

---

## 5. Pool Trust Model

**Trust is NOT transitive across pools.** Being in Pool A does not grant any trust from Pool B.

**Cross-pool cross-domain diversity**: when a path spans multiple pool segments across multiple domains, operator diversity is enforced across the ENTIRE path including domain bridge nodes. A single operator CANNOT appear in multiple segments even if those segments are in different routing domains.

**Mesh island pools**: pools scoped to a mesh island contain only nodes with mesh endpoints in that island. Clients selecting from mesh island pools verify each node has a valid mesh island endpoint.

---

## 6. Pool Member Verification

For curated pools, the curator signs each member's inclusion separately:

```json
{
  "pool_id": "...",
  "node_id": "...",
  "added_at": "2026-04-01T00:00:00Z",
  "added_by": "<curator_public_key base64url>",
  "member_signature": "<Ed25519 signature>"
}
```

Member signature signs: `BLAKE3("bgpx-pool-member-v1" || pool_id || node_id || added_at_timestamp_BE8)`

---

## 7. Pool Lifecycle

```
Curator generates Ed25519 keypair (curator key)
     │
     ▼
Curator creates pool advertisement (including domain_scope if applicable)
     │
     ▼
Curator publishes to DHT (DHT_PUT)
     │
     ├── Every 3 days: re-publish with updated timestamps
     ├── On member change: re-publish with updated member list
     ├── On domain_scope change: re-publish immediately
     └── On shutdown: pool expires naturally after 7 days
```

---

## 8. Empty Pool Handling

```python
if pool_active_members < required_hops:
    if domain_scope != "any":
        # Domain-scoped pool: only fall back to default pool within same domain
        default_candidates = get_nodes_in_domain("bgpx-default", domain)
        if len(default_candidates) >= required_hops:
            use_pool = "bgpx-default"
        else:
            return FAIL(ERR_POOL_INSUFFICIENT_NODES)
    else:
        if fallback_to_default:
            use_pool = "bgpx-default"
        else:
            return FAIL(ERR_POOL_INSUFFICIENT_NODES)
```

---

## 9. Pool Security Considerations

**Curator compromise**: adversary can sign fake pool advertisements and include malicious nodes. Individual node verification still catches nodes with invalid individual advertisements. Cross-pool diversity limits damage to one segment. Emergency rotation with 2-hour window available.

**Pool Sybil**: adversary operates many real nodes and convinces curator to include them. Reputation system degrades misbehaving nodes. Weighted random selection limits dominance. Cross-domain operator diversity limits per-operator presence.

**Domain bridge pool specific**: adversary fills a domain-bridge pool with bridge nodes they control. Individual node verification requires each node to actually serve both domains. Operator diversity prevents single adversary controlling all bridge positions.

---

## 10. Pool Curator Key Rotation

See `/protocol/pool_curator_key_rotation.md` for full specification. Domain-scoped pools and domain-bridge pools use the same rotation mechanism.

---

## 11. Pool-Related Error Codes

| Code | Name | Description |
|---|---|---|
| 0x0011 | ERR_POOL_NOT_FOUND | Pool ID not found in DHT |
| 0x0012 | ERR_POOL_SIGNATURE_INVALID | Pool advertisement signature verification failed |
| 0x0013 | ERR_POOL_INSUFFICIENT_NODES | Pool has fewer active nodes than required |
| 0x0014 | ERR_POOL_ROLE_MISMATCH | Pool has no nodes with the required role for this position |
| 0x0015 | ERR_SEGMENT_CONSTRUCTION_FAILED | Path segment could not be constructed |
| 0x0023 | ERR_DOMAIN_SCOPE_MISMATCH | Pool domain_scope does not match required segment domain |
| 0x0024 | ERR_DOMAIN_BRIDGE_POOL_INVALID | Domain-bridge pool does not match required bridge pair |
