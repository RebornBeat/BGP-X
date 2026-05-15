# BGP-X DHT Pool Specification

**Version**: 0.1.0-draft

This document is the authoritative specification for BGP-X DHT Pools — named, signed collections of nodes that enable multi-tier trust paths, double-exit architecture, domain-scoped routing, and pool-based path construction.

---

## 1. Overview

A DHT Pool is a named, signed collection of BGP-X nodes grouped by trust level, operational purpose, or routing domain. Pools enable path construction that routes different segments of a path through different trusted node groups.

Pools do NOT replace individual node verification. Pool membership is a reputation signal and selection mechanism — each pool member's individual advertisement is still verified independently.

---

## 2. Pool Types

| Type | Description | Membership Discovery | Who Can Join |
|---|---|---|---|
| Public | Open collection of nodes that opt in | DHT lookup | Any node (self-attestation) |
| Curated | Curator vets and signs membership | DHT lookup | Nodes approved by curator |
| Private | Membership list shared out-of-band | Out-of-band | By invitation only |
| Semi-private | Pool ID discoverable, join requires approval | DHT lookup | By curator approval |
| Ephemeral | Session-scoped, not published to DHT | Direct communication | Client-specified |
| Domain-scoped | Restricted to nodes in a specific routing domain | DHT lookup with domain filter | Domain-eligible nodes |
| Domain-bridge | Composed of bridge nodes for a specific domain pair | DHT lookup | Bridge-capable nodes only |

### Satellite-Connected Nodes in Pools

Satellite internet services (Starlink, Iridium, etc.) provide clearnet connectivity. Satellite-connected nodes with BGP-X capability are clearnet domain nodes and appear in clearnet domain-scoped pools. There is no "satellite domain" pool type — satellite is a clearnet transport variant.

---

## 3. Pool Structure

### 3.1 Pool Advertisement JSON

```json
{
  "version": 1,
  "pool_id": "<BLAKE3 hash, hex, 32 bytes>",
  "pool_name": "EU No-Log Exit Nodes",
  "pool_type": "curated",
  "curator_public_key": "<Ed25519 public key, base64url>",
  "member_list_format": "embedded",
  "members": [
    "node_id_1_hex",
    "node_id_2_hex"
  ],
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

### 3.2 Field Definitions

| Field | Required | Description |
|---|---|---|
| version | REQUIRED | Always 1 |
| pool_id | REQUIRED | BLAKE3(curator_public_key \|\| pool_name_utf8) |
| pool_name | REQUIRED | Human-readable name |
| pool_type | REQUIRED | "public", "curated", "private", "semi_private", "ephemeral", "domain_scoped", "domain_bridge" |
| curator_public_key | REQUIRED | Ed25519 public key of curator |
| member_list_format | REQUIRED | "embedded" (≤100 members) or "external" (large pools) |
| members | CONDITIONAL | Required if member_list_format = "embedded" |
| member_count | REQUIRED | Total member count |
| description | OPTIONAL | Human-readable description |
| domain_scope | OPTIONAL | "any" (default), "clearnet", "mesh:<island_id>", or "lora-regional:<region_id>" |
| bridges | CONDITIONAL | Required if pool_type = "domain_bridge"; specifies bridge pair |
| min_reputation_for_selection | OPTIONAL | Minimum reputation for selection within this pool |
| regions_covered | OPTIONAL | Regions where pool has coverage |
| min_uptime_pct | OPTIONAL | Minimum uptime for pool members |
| exit_policies_required | OPTIONAL | Whether all members must be exit nodes |
| exit_logging_policy_required | OPTIONAL | Required logging policy for exit members |
| signed_at | REQUIRED | ISO 8601 UTC |
| expires_at | REQUIRED | ISO 8601 UTC, max 7 days after signed_at |
| signature | REQUIRED | Ed25519 signature over canonical JSON |

### 3.3 Field: domain_scope

Restricts pool membership to nodes in a specific routing domain.

Values:
- `"any"` (default): pool contains nodes from any routing domain — backward compatible with all existing pool usage
- `"clearnet"`: pool contains only clearnet-reachable nodes
- `"mesh:<island_id>"`: pool contains only nodes in the specified mesh island
- `"lora-regional:<region_id>"`: pool contains only nodes in the specified LoRa region

### 3.4 Field: bridges (for domain-bridge pools)

When `pool_type = "domain_bridge"`, describes the specific bridge pair this pool serves:

```json
{
  "pool_type": "domain_bridge",
  "bridges": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "from_domain_name": "clearnet",
      "to_domain_name": "mesh:lima-district-1",
      "bridge_latency_ms": 25,
      "transport": "wifi_mesh",
      "available": true
    }
  ]
}
```

A domain-bridge pool contains only nodes that bridge the specified domain pair. Each member's individual advertisement MUST confirm it serves both domains. Pool membership alone is insufficient — individual node advertisement verification REQUIRED.

### 3.5 Member List Format: External

For large pools (>100 members), the member list is served from an external endpoint:

```json
{
  "member_list_format": "external",
  "member_endpoint": "https://pool-registry.example.com/pools/pool-id/members",
  "member_endpoint_signature_key": "<Ed25519 public key, base64url>",
  ...
}
```

The external endpoint returns:
```json
{
  "pool_id": "...",
  "members": ["node_id_1", "node_id_2", ...],
  "signed_at": "...",
  "signature": "..."
}
```

The member list response is separately signed with `member_endpoint_signature_key`.

---

## 4. Pool Discovery Protocol

### 4.1 DHT Storage Key

```
pool_key = BLAKE3("bgpx-pool-v1" || pool_id)
TTL: 7 days maximum. Re-publication: every 3 days.
```

### 4.2 Unified DHT Storage

Pool advertisements are stored in the **unified BGP-X DHT** — the same DHT that stores node advertisements, domain bridge records, and mesh island records. There is no separate pool DHT.

The DHT key derivation uses a prefix to avoid collision with other record types:
- Pool advertisement: `BLAKE3("bgpx-pool-v1:" || pool_id)`
- Node advertisement: `BLAKE3("node_adv_v1:" || node_id)`
- Domain bridge: `BLAKE3("bgpx-domain-bridge-v1:" || domain_a_id || domain_b_id)`
- Mesh island: `BLAKE3("bgpx-mesh-island-v1:" || island_id)`

All prefixes ensure distinct key spaces within the unified DHT.

### 4.3 Publishing

Pool curators publish advertisements via DHT_PUT to the pool_key. Storage nodes verify the advertisement signature before storing.

Pool TTL: 7 days maximum. Curators re-publish every 3 days.

### 4.4 Retrieval

Clients retrieve pool advertisements via DHT_GET on the pool_key. On receipt:

1. Verify advertisement signature against curator_public_key
2. Verify pool_id = BLAKE3(curator_public_key || pool_name_utf8)
3. Verify timestamps are valid (not expired, not future)
4. Cache the pool advertisement locally (TTL from expires_at)

### 4.5 Pool Advertisement Signature Verification

```
canonical = canonical_json(advertisement, exclude="signature")
valid = ed25519_verify(
    public_key = curator_public_key,
    signature = advertisement.signature,
    message = canonical.encode("utf-8")
)
```

Canonical JSON: UTF-8, compact, keys sorted alphabetically, signature field excluded.

### 4.6 Domain-Scoped Pool Discovery

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

## 5. Pool Membership Verification

### 5.1 Per-Node Individual Verification (MANDATORY)

Regardless of pool membership, each node's own advertisement MUST be independently verified:

```
1. Fetch node advertisement from DHT (by NodeID)
2. Verify advertisement signature against node's own public_key
3. Verify node_id = BLAKE3(public_key)
4. Verify timestamps (not expired, not future)
5. Verify advertisement not withdrawn (withdrawn = false)
6. Verify node not locally blacklisted
7. If pool has role requirements: verify node has required roles
```

Pool membership alone does NOT bypass individual verification.

### 5.2 Pool Member Signature

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

**Member Signature Canonical Form**:

The member signature canonical form is:

```
canonical_member = "bgpx-pool-member-v1:" || pool_id_hex || ":" || node_id_hex || ":" || added_at_unix_timestamp_BE8
signature = Ed25519Sign(curator_private_key, BLAKE3(canonical_member))
```

Example:
```
canonical_member = "bgpx-pool-member-v1:abc123...:def456...:17119296000000000000"
```

The signature proves the curator included this specific node at this specific time. Member signatures are OPTIONAL — pools may choose to include them for additional provenance verification.

### 5.3 Geographic Plausibility and Pool Membership

Pool membership is independent of geographic plausibility scoring.

- A node may be in a pool without declaring jurisdiction
- Geographic plausibility scoring only applies to nodes that voluntarily declare jurisdiction in their individual advertisement
- Pool membership does NOT require jurisdiction declaration
- Pool `regions_covered` field is informational metadata — it does not enforce or require geo plausibility participation

Satellite-connected nodes in clearnet pools are exempt from geo plausibility scoring regardless of jurisdiction declaration.

---

## 6. Pool Trust Model

### 6.1 Trust is NOT Transitive

Pool A trusting node X does not mean Pool B trusts node X. Each pool defines its own membership independently.

### 6.2 Pool Trust Isolation

When a path uses multiple segments from different pools:

- Nodes in Segment 1 (Pool A) cannot see Segment 2 (Pool B) nodes, and vice versa
- The path_id is opaque to nodes outside their segment
- Compromise of Pool A curator cannot affect Pool B membership

### 6.3 Cross-Pool Cross-Domain Diversity Enforcement

When a path spans multiple pool segments across multiple domains, operator diversity is enforced across the ENTIRE path including domain bridge nodes. A single operator CANNOT appear in multiple segments even if those segments are in different routing domains.

### 6.4 Mesh Island Pools

Pools scoped to a mesh island contain only nodes with mesh endpoints in that island. Clients selecting from mesh island pools verify each node has a valid mesh island endpoint.

---

## 7. Pool Anonymization Strategies

### 7.1 Trust Tier Path

Use different pools for different hops based on trust level:

```
[Untrusted public pool entry] → [Curated trusted relays] → [Private self-operated exit]
```

Even if the first segment is compromised (adversary controls entry), the subsequent segments provide separation.

### 7.2 Double-Exit Architecture

Two independent exit pools in sequence:

```
[Entry + Relay from default pool] → [Exit from Pool 1] → [Exit from Pool 2] → Destination
```

- Pool 1 exit sees: traffic going to Pool 2 exit (not the final destination)
- Pool 2 exit sees: traffic from Pool 1 exit (not the original client path)
- No single exit operator can see both path origin and final destination

Double-exit is particularly valuable when:
- Accessing high-value targets (banking, government services)
- Operating from a jurisdiction with aggressive surveillance
- The destination service is itself a potential adversary

### 7.3 Pool Portfolio Rotation

Client randomly selects from multiple curated pools per connection:

```
Available pools: [pool-eu-a, pool-eu-b, pool-na-a]
Per connection: randomly select one
```

Prevents analysis of which specific pool nodes always appear in paths.

### 7.4 Geographic Scatter Strategy

Force path to cross multiple distinct geographic regions:

```
Segment 1: [Pool: na-relays]   → entry in North America
Segment 2: [Pool: eu-relays]   → relay in Europe
Segment 3: [Pool: ap-exits]    → exit in Asia-Pacific
```

Global passive adversary must monitor all three geographic regions simultaneously.

### 7.5 Cross-Domain Trust Separation

For paths that cross routing domains:

```
Segment 1: [Clearnet pool] → DOMAIN_BRIDGE → Segment 2: [Mesh island pool]
```

- Clearnet pool nodes cannot see mesh domain segment
- Mesh pool nodes cannot see clearnet segment
- Domain bridge node sees only the transition point, not content

This provides automatic trust separation across routing domain boundaries.

---

## 8. Pool Lifecycle

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

### Mesh Island Grace Period Handling

If a mesh island is disconnected from the clearnet during a pool's grace period:

1. Bridge nodes cache the rotation record
2. When the island reconnects, bridge nodes propagate the rotation to mesh nodes
3. Grace period countdown pauses while island is offline (honest implementation)
4. Alternatively: curator may extend grace period when scheduling rotation during expected island downtime

Curators should avoid scheduling rotations during expected island maintenance windows.

---

## 9. Empty Pool Handling

When a pool has insufficient active nodes for the requested number of hops:

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

Client configuration option:
```toml
[path_constraints]
fallback_to_default = true   # Allow fallback when pool is insufficient
min_pool_members = 5         # Minimum acceptable pool size
```

---

## 10. Domain-Bridge Pool Verification

Domain-bridge pools (`pool_type = "domain_bridge"`) require additional verification:

1. Pool advertisement MUST include `bridges` field with `from_domain` and `to_domain`
2. Each listed member MUST have `bridge_capable = true` in their individual advertisement
3. Each member's `bridges` array MUST include the pool's bridge pair
4. Each member MUST have active endpoints in both domains

Verification procedure:
```
for member in pool.members:
    node_ad = fetch_node_advertisement(member)
    if not node_ad.bridge_capable:
        reject(member, "Node not bridge_capable")
    if (from_domain, to_domain) not in node_ad.bridges:
        reject(member, "Node does not serve this bridge pair")
    if not has_endpoint_in_domain(node_ad, from_domain):
        reject(member, "Node missing endpoint in from_domain")
    if not has_endpoint_in_domain(node_ad, to_domain):
        reject(member, "Node missing endpoint in to_domain")
    accept(member)
```

A domain-bridge pool with invalid members returns ERR_DOMAIN_BRIDGE_POOL_INVALID.

---

## 11. Pool Security Considerations

### 11.1 Curator Compromise

If a pool curator's private key is compromised:

1. Adversary can sign fake pool advertisements and fake member inclusions
2. Individual node verification still catches: adversary cannot forge individual node signatures
3. Adversary CAN include their own malicious nodes in the pool advertisement
4. Defense: clients who observe pool-level anomalies can report and avoid the pool

Mitigation for operators: use pool curator key rotation immediately upon discovering compromise.

### 11.2 Pool Sybil Attack

Adversary creates many real BGP-X nodes and convinces a curator to include them:

- Individual node signature requirement: adversary must operate real nodes
- Reputation system: new nodes start at 60.0 and must earn higher scores
- Cross-pool operator diversity: adversary nodes from same operator limited in single path
- Weighted random selection: Sybil nodes compete with legitimate nodes probabilistically

### 11.3 Pool Capture Attack

Adversary gradually introduces malicious members to a curated pool:

- Per-node individual verification catches nodes with poor reputation
- Pool minimum reputation setting (`min_reputation_for_selection`) filters low-reputation nodes
- Reputation system degrades malicious nodes over time
- Cross-pool diversity limits damage even if one pool is heavily compromised

### 11.4 Domain Bridge Pool Specific

Adversary fills a domain-bridge pool with bridge nodes they control. Individual node verification requires each node to actually serve both domains. Operator diversity prevents single adversary controlling all bridge positions.

---

## 12. Pool Curator Key Rotation

See `/protocol/pool_curator_key_rotation.md` for the complete key rotation specification.

Summary:

- Pool curators may rotate their signing key using a dual-signature rotation record
- Rotation records require both old key and new key to sign
- Emergency rotation uses 2-hour acceptance window
- After rotation, all pool members signed under old key have a 24-hour grace period
- Curator re-signs all members within 24 hours as part of rotation

Domain-scoped pools and domain-bridge pools use the same rotation mechanism.

---

## 13. Pool-Related Error Codes

| Code | Name | Description |
|---|---|---|
| 0x0011 | ERR_POOL_NOT_FOUND | Pool ID not found in DHT |
| 0x0012 | ERR_POOL_SIGNATURE_INVALID | Pool advertisement signature verification failed |
| 0x0013 | ERR_POOL_INSUFFICIENT_NODES | Pool has fewer active nodes than required |
| 0x0014 | ERR_POOL_ROLE_MISMATCH | Pool has no nodes with the required role for this position |
| 0x0015 | ERR_SEGMENT_CONSTRUCTION_FAILED | Path segment could not be constructed |
| 0x0023 | ERR_DOMAIN_SCOPE_MISMATCH | Pool domain_scope does not match required segment domain |
| 0x0024 | ERR_DOMAIN_BRIDGE_POOL_INVALID | Domain-bridge pool does not match required bridge pair |
