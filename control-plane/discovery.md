# BGP-X Node Discovery Specification

**Version**: 0.1.0-draft

---

## 1. Overview

BGP-X uses a **single unified Distributed Hash Table (DHT)** for node discovery. All routing domains — clearnet, mesh islands, satellite — participate in the same Kademlia key space.

**Previous model (replaced)**: two separate DHTs (internet DHT + mesh DHT) with gateway synchronization.

**Current model**: one unified DHT. Domain bridge nodes serve as the physical infrastructure for DHT storage accessible from the internet. Mesh-only nodes access the unified DHT via their bridge nodes' caches. There is no synchronization between separate DHTs because there is only one DHT.

---

## 2. DHT Algorithm

Kademlia-style DHT. XOR distance metric. k = 20 bucket size. α = 3 parallel queries. O(log N) lookup.

NodeID = BLAKE3(Ed25519_public_key) — same for all routing domains.

---

## 3. Record Types and Storage Keys

### Node Advertisement
```
key = BLAKE3("bgpx-node-advert-v1" || node_id)
TTL: 48 hours. Re-publication: every 12 hours.
```

### Pool Advertisement
```
key = BLAKE3("bgpx-pool-v1" || pool_id)
TTL: 7 days. Re-publication: every 3 days.
```

### Domain Bridge Advertisement
```
key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id, to_domain_id) || max(from_domain_id, to_domain_id))
TTL: 24 hours. Re-publication: every 8 hours.
Symmetric: A→B and B→A use same key.
```

### Mesh Island Advertisement
```
key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8_bytes)
TTL: 24 hours. Re-publication: every 8 hours.
```

### Pool Curator Key Rotation Record
```
key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || rotation_timestamp_BE8)
TTL: accept_until + 7 days
```

---

## 4. DHT Storage Authentication

Nodes accepting DHT_PUT MUST verify before storing:

```python
function accept_put(record, storage_key):

    advert = parse_record(record.payload)

    # Verify signature
    canonical = canonical_json(advert, exclude="signature")
    if not ed25519_verify(advert.public_key, advert.signature, canonical.encode("utf-8")):
        drop_silently("Invalid signature")
        return

    # For node advertisements: verify NodeID
    if advert.type == "node_advertisement":
        if advert.node_id != hex(BLAKE3(base64url_decode(advert.public_key))):
            drop_silently("NodeID mismatch")
            return

    # For domain bridge advertisements: verify node serves both bridged domains
    if advert.type == "domain_bridge_advertisement":
        # Fetch node's own advertisement from DHT cache
        node_advert = get_from_dht_cache(advert.node_id)
        if node_advert:
            if not node_advert.bridge_capable:
                drop_silently("Non-bridge node claiming bridge capability")
                return

    # Verify timestamps
    if advert.expires_at <= current_time():
        drop_silently("Expired advertisement")
        return

    # Accept and store
    storage.put(storage_key, record.payload, ttl=advert.expires_at - current_time())
```

---

## 5. Domain-Filtered DHT Queries

DHT_FIND_NODE extended with optional domain filter:

```
Standard DHT_FIND_NODE payload:
  Target NodeID (32 bytes)

Extended DHT_FIND_NODE payload:
  Target NodeID (32 bytes)
  Domain filter flag (1 byte): 0x00 = no filter, 0x01 = filter active
  Domain ID (8 bytes): only present if flag = 0x01
```

When filter active: storage nodes return only contacts with advertised endpoints in the specified routing domain.

---

## 6. Bootstrap Process

### Internet Mode

New nodes bootstrap via hardcoded list → DHT_FIND_NODE → populate routing table → publish own advertisement.

DNS fallback: `_bgpx-bootstrap._udp.bgpx.network` TXT records (queried via DoH).

Additional bootstrap step for cross-domain:
```
After DHT routing table populated:
  → Query for DOMAIN_ADVERTISE records for needed bridge pairs
  → Query for MESH_ISLAND_ADVERTISE for known island IDs
  → Cache bridge node and island records for path construction
```

### Mesh-Only Mode

Bootstrap via MESH_BEACON broadcast. No hardcoded internet bootstrap nodes needed. Bootstrap complete when routing table has ≥ 5 entries from ≥ 2 distinct nodes.

After local DHT populated, mesh nodes can access unified internet DHT via their bridge nodes:
- Mesh node sends DHT query to bridge node via mesh transport
- Bridge node executes query against unified internet DHT
- Returns results via mesh transport

### Bootstrap for Cross-Domain Discovery

When a mesh island wants to discover another island or clearnet nodes:
1. Mesh node queries local bridge node for MESH_ISLAND_ADVERTISE of target island
2. Bridge node queries unified internet DHT if not in local cache
3. Mesh node uses returned island advertisement to find bridge nodes for target island
4. Path construction proceeds

---

## 7. Domain Bridge Discovery

```python
function discover_bridge_nodes(from_domain, to_domain):

    # Compute symmetric bridge key
    sorted_pair = sorted([from_domain.id_bytes, to_domain.id_bytes])
    bridge_key = BLAKE3("bgpx-domain-bridge-v1" || sorted_pair[0] || sorted_pair[1])

    bridge_record = dht_get(bridge_key)
    if bridge_record is None:
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    if not verify_bridge_record_signature(bridge_record):
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    verified_bridges = []
    for node_id in bridge_record.bridge_nodes:
        node_advert = dht_get(BLAKE3("bgpx-node-advert-v1" || node_id))
        if node_advert and verify_node_advertisement(node_advert):
            if node_advert.bridge_capable and serves_both_domains(node_advert, from_domain, to_domain):
                verified_bridges.append(node_advert)

    if not verified_bridges:
        return FAIL(ERR_DOMAIN_BRIDGE_UNAVAILABLE)

    return verified_bridges
```

---

## 8. Unified DHT Participation by Mesh-Only Nodes

**Publishing**:
- Mesh node constructs its node advertisement
- Sends to bridge node via mesh transport
- Bridge node publishes to unified internet DHT via DHT_PUT
- Mesh node's advertisement is globally discoverable

**Querying**:
- Mesh node sends DHT query to bridge node via mesh transport
- Bridge node executes query against unified internet DHT
- Returns results via mesh transport

**Offline mode** (no bridge node):
- Mesh node participates only in local mesh DHT (via MESH_BEACON bootstrap)
- Advertisement not globally discoverable
- When bridge reconnects: pending mesh advertisement published to unified DHT

---

## 9. Routing Table Maintenance

**Bucket refresh**: any k-bucket not queried in last 60 minutes → DHT_FIND_NODE for random key in that range.

**Dead node removal**: fails 3 consecutive queries over 5 minutes → removed from routing table.

**Stale advertisement pruning**: storage nodes MUST prune expired advertisements.

---

## 10. Geographic Plausibility Integration

DHT routing table maintenance includes RTT measurement per node. Domain-specific thresholds applied. RTT measurements feed the geographic plausibility scoring system.

---

## 11. Pool Discovery

Standard pool discovery: `DHT_GET(BLAKE3("bgpx-pool-v1" || pool_id))`.

Domain-scoped pool discovery: after retrieval, verify `pool.domain_scope` matches required segment domain.

Domain-bridge pool discovery: after retrieval, verify `pool.bridges` matches required domain transition pair.

---

## 12. Island Collision Detection

When a mesh island registers island_id in the unified DHT:
1. Perform DHT_GET for island_key before publishing
2. If record exists from different operator: collision detected; choose different island_id
3. island_id should be descriptive and geographically specific (e.g., `lima-san-isidro-2026` not `community-1`)

---

## 13. DHT Privacy Considerations

**Query routing through overlay**: once a client has established any BGP-X path, all subsequent DHT queries SHOULD be routed through that path. Domain-filtered queries for mesh island nodes reveal interest in that island to DHT storage nodes.

**Query batching**: fetch multiple bridge records and island records in one path traversal.

**Pre-fetch**: commonly needed bridge records (for frequently used domain transitions) pre-fetched before path construction events.

**Mesh island privacy**: islands that want to remain undiscoverable from clearnet should not publish MESH_ISLAND_ADVERTISE (operate in offline mode only, accessible via known bridge node only).
