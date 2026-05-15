# BGP-X Node Discovery Specification

**Version**: 0.1.0-draft

This document specifies how BGP-X nodes discover each other and how clients find available relay nodes for path construction.

---

## 1. Overview

BGP-X uses a **single unified Distributed Hash Table (DHT)** for node discovery. All routing domains — clearnet, mesh islands, satellite-connected nodes — participate in the same Kademlia key space.

### Design Properties

- **Fully decentralized**: No central server, no directory authority, no single point of failure or coercion
- **Self-organizing**: Nodes join and leave without coordination
- **Cryptographically authenticated**: All records are signed; unsigned or invalid records are rejected
- **Privacy-preserving**: DHT queries for nodes are distinguishable from path-construction traffic only by an adversary who can observe both the query and the response at the network level
- **Sybil-resistant**: The cost of populating the DHT with fake nodes is bounded by the signature verification requirement and the reputation system
- **Transport-agnostic**: Operates over UDP/IP for internet, mesh beacon protocol for offline mesh

### Unified DHT Architecture

**Previous model (replaced)**: two separate DHTs (internet DHT + mesh DHT) with gateway synchronization.

**Current model**: one unified DHT. Domain bridge nodes serve as the physical infrastructure for DHT storage accessible from the internet. Mesh-only nodes access the unified DHT via their bridge nodes' caches. There is no synchronization between separate DHTs because there is only one DHT.

---

## 2. DHT Algorithm

BGP-X uses a **Kademlia**-style DHT. Kademlia is well-studied, widely deployed (used in BitTorrent, IPFS, Ethereum), and provides efficient O(log N) lookup in networks of N nodes.

### 2.1 Key Space

The DHT key space is 256 bits, matching the output length of BLAKE3.

All DHT keys are 256-bit values. Node positions in the DHT are determined by their NodeID:

```
node_position = NodeID = BLAKE3(Ed25519_public_key)
```

NodeID is derived identically for all routing domains. A node has one NodeID regardless of which routing domains it participates in.

### 2.2 Distance Metric

Kademlia uses XOR as a distance metric:

```
distance(a, b) = a XOR b
```

This metric is symmetric and satisfies the triangle inequality, making it suitable for routing decisions.

### 2.3 Routing Table (k-bucket structure)

Each node maintains a routing table organized into **k-buckets**:

- The key space is divided into 256 buckets (one per bit of the key space)
- Each bucket covers a range of the key space at a specific distance from the node's own ID
- Each bucket holds at most **k** node contacts (BGP-X default: k = 20)
- Buckets are sorted by last-seen time (most recently seen first)

When a bucket is full and a new node is discovered for that bucket's range:

1. Ping the least-recently-seen node in the bucket
2. If it responds: keep it, discard the new node (stable nodes are preferred)
3. If it does not respond: replace it with the new node

This replacement policy naturally favors nodes with high uptime — a Sybil attacker who floods the network with ephemeral nodes will not displace well-established nodes.

### 2.4 Lookup Algorithm

To find nodes near a target key:

1. Select the α closest known nodes to the target (BGP-X default: α = 3)
2. Send parallel DHT_FIND_NODE queries to all α nodes
3. From the responses, identify the k closest nodes found so far
4. Send queries to any newly discovered nodes that are closer than the current k-closest
5. Repeat until no closer nodes are discovered
6. Return the k closest nodes found

BGP-X uses α = 3 (3 parallel queries) as the default concurrency parameter. Higher α values speed up lookups at the cost of more network traffic.

---

## 3. Record Types and Storage Keys

### 3.1 Node Advertisement

```
storage_key = BLAKE3("bgpx-node-advert-v1" || node_id)
TTL: 48 hours
Re-publication: every 12 hours
```

Node advertisements are stored in the DHT and retrieved by clients during path construction.

### 3.2 Pool Advertisement

```
storage_key = BLAKE3("bgpx-pool-v1" || pool_id)
TTL: 7 days
Re-publication: every 3 days
```

Pool advertisements are published by pool curators and discovered by clients constructing paths.

### 3.3 Domain Bridge Advertisement

```
storage_key = BLAKE3("bgpx-domain-bridge-v1" || min(from_domain_id, to_domain_id) || max(from_domain_id, to_domain_id))
TTL: 24 hours
Re-publication: every 8 hours
```

**Symmetric ordering**: A→B and B→A use the same key. Bridge discovery is bidirectional.

Storage nodes MUST verify DOMAIN_ADVERTISE signatures before storing. The advertisement is signed by the node's private key (same Ed25519 keypair as node advertisement).

**Important**: A bridge node must have verified endpoints in BOTH domains it claims to bridge. Storage nodes should verify this by checking the node advertisement's `routing_domains` field.

### 3.4 Mesh Island Advertisement

```
storage_key = BLAKE3("bgpx-mesh-island-v1" || island_id_utf8_bytes)
TTL: 24 hours
Re-publication: every 8 hours
```

Mesh island gateway nodes publish island advertisements to the unified DHT when internet-connected.

When no bridge node with internet is available, the island advertisement exists only in the local mesh DHT. When a bridge node reconnects, the island advertisement is published to the unified internet DHT automatically.

### 3.5 Pool Curator Key Rotation Record

```
storage_key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || rotation_timestamp_BE8)
TTL: accept_until + 7 days
```

Key rotation records are stored in the unified DHT for clients to verify pool membership signatures during the grace period.

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

Implementations that accept unsigned or invalid records into DHT storage are non-compliant.

---

## 5. Node Advertisement Publication

A node publishes its advertisement by:

1. Constructing and signing the advertisement (see `/control-plane/node_advertisement.md`)
2. Computing the storage key
3. Finding the k nodes closest to the storage key via DHT lookup
4. Sending DHT_PUT to each of those k nodes

The k closest nodes to the storage key are responsible for storing the advertisement and serving it to requesters.

---

## 6. Node Advertisement Retrieval

A client retrieves a node's advertisement by:

1. Computing the storage key from the known NodeID
2. Finding the k nodes closest to the storage key
3. Sending DHT_GET to those nodes
4. Receiving the signed advertisement record
5. Verifying the signature and expiry before using the record

---

## 7. Re-publication Requirements

Node advertisements expire after a maximum of 48 hours (set in the `expires_at` field). Nodes MUST re-publish their advertisement before it expires.

**Recommended re-publication interval: every 12 hours**

When re-publishing, nodes MUST update the `signed_at` and `expires_at` timestamps and re-sign the advertisement.

Nodes that fail to re-publish within 48 hours will disappear from the DHT naturally as their records expire on storage nodes.

Pool advertisement TTL: 7 days. Re-publication: every 3 days.

Domain bridge advertisement TTL: 24 hours. Re-publication: every 8 hours.

Mesh island advertisement TTL: 24 hours. Re-publication: every 8 hours.

---

## 8. Node Withdrawal from DHT

When a node shuts down gracefully, it publishes NODE_WITHDRAW:

1. Node constructs NODE_WITHDRAW message (NodeID + timestamp + signature)
2. Sends to k storage nodes closest to its storage key
3. Storage nodes remove the advertisement immediately upon receiving valid withdrawal
4. Storage nodes propagate the withdrawal to 5 nearest DHT peers

**Propagation limit**: Withdrawal propagates to 5 nearest peers, then stops. This prevents infinite gossip amplification.

**Natural fallback**: If NODE_WITHDRAW is not received (crash, power failure), the advertisement expires within 48 hours.

For domain bridge nodes: after NODE_WITHDRAW, also remove from bridge pair records.

Nodes MUST publish NODE_WITHDRAW before graceful shutdown.

---

## 9. Pool Discovery Protocol

Pool discovery uses the same DHT infrastructure as node discovery.

### 9.1 Pool Advertisement Retrieval

```
1. Compute pool_key = BLAKE3("bgpx-pool-v1" || pool_id)
2. DHT_GET(pool_key)
3. Verify pool advertisement signature against curator_public_key
4. Verify pool_id matches derived value
5. Cache pool advertisement (TTL from expires_at)
```

### 9.2 Pool Member List Retrieval

**Embedded format (≤100 members)**: members included in pool advertisement JSON.

**External format (>100 members)**:
1. Pool advertisement contains `member_endpoint` URL
2. Client fetches member list from URL (optionally over BGP-X overlay)
3. Verify member list signature against `member_endpoint_signature_key` from pool advertisement
4. Cache member list

### 9.3 Domain-Scoped Pool Discovery

After retrieval, verify `pool.domain_scope` matches required segment domain.

### 9.4 Domain-Bridge Pool Discovery

After retrieval, verify `pool.bridges` matches required domain transition pair.

---

## 10. Domain-Filtered DHT Queries

DHT_FIND_NODE is extended to support optional domain filtering:

```
Standard DHT_FIND_NODE payload:
  Target NodeID (32 bytes)

Extended DHT_FIND_NODE payload:
  Target NodeID (32 bytes)
  Domain filter flag (1 byte): 0x00 = no filter, 0x01 = filter active
  Domain ID (8 bytes): only present if flag = 0x01
```

When domain filter is active: storage nodes MUST return only contacts with advertised endpoints in the specified routing domain.

---

## 11. Bootstrap Process

### 11.1 Internet Mode (IP-based)

New nodes bootstrap via hardcoded bootstrap node list:

```
1. Node starts with hardcoded bootstrap node list
2. For each bootstrap node:
   a. Send DHT_FIND_NODE(target = own NodeID)
   b. Receive k closest nodes to own NodeID
   c. Add discovered nodes to routing table
3. Send DHT_FIND_NODE to newly discovered nodes
   (iterative lookup converging on own NodeID)
4. Routing table is now populated
5. Publish own advertisement via DHT_PUT
6. Node is now a full DHT participant
```

**DNS-based fallback**: `_bgpx-bootstrap._udp.bgpx.network` TXT records. Query via DoH for this lookup to prevent ISP interception.

**Additional bootstrap step for cross-domain**:
```
After DHT routing table populated:
  → Query for DOMAIN_ADVERTISE records for needed bridge pairs
  → Query for MESH_ISLAND_ADVERTISE for known island IDs
  → Cache bridge node and island records for path construction
```

### 11.2 Mesh-Only Mode (No Internet)

New mesh nodes bootstrap via broadcast beacons:

1. Broadcast MESH_BEACON on all active mesh transports
2. Receive MESH_BEACONs from nearby nodes
3. Verify beacon signatures
4. Add beaconing nodes to DHT routing table
5. Send DHT_FIND_NODE via mesh transport to discovered nodes
6. Publish own advertisement to mesh DHT
7. Node is a full mesh DHT participant

No hardcoded bootstrap nodes needed. Bootstrap completes when routing table has ≥ 5 entries from ≥ 2 distinct nodes.

After local DHT populated, mesh nodes can access unified internet DHT via their bridge nodes:
- Mesh node sends DHT query to bridge node via mesh transport
- Bridge node executes query against unified internet DHT
- Returns results via mesh transport

### 11.3 Bootstrap Node Properties

Bootstrap nodes (internet mode):
- Run by the BGP-X project or trusted community operators
- High uptime SLA (99.9%+)
- Geographically distributed across multiple continents and jurisdictions
- Do NOT serve as relay or exit nodes (they serve DHT only)
- Hardcoded NodeID + public key in software (immutable identity)
- DDoS protection recommended (10 Gbps+ scrubbing)

**Bootstrap nodes are not trusted authorities.** They are entry points into the DHT graph. Once a joining node has populated its routing table, bootstrap nodes are no longer needed for routing.

### 11.4 Bootstrap Node Failure Tolerance

If fewer than 3 bootstrap nodes are reachable:

- Log a warning
- Retry after 30 seconds (exponential backoff up to 5 minutes)
- Continue attempting to join until successful

If no bootstrap nodes are reachable after 15 minutes:

- Log an error
- Enter BOOTSTRAP_FAILED state
- Alert operator

BGP-X clients that cannot reach any bootstrap node MUST NOT attempt to use the network (they cannot verify they have valid node data).

---

## 12. Domain Bridge Discovery

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

## 13. Unified DHT Participation by Mesh-Only Nodes

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

## 14. Cross-Domain Discovery

### 14.1 Mesh Island Discovering Another Island or Clearnet

When a mesh island wants to discover another island or clearnet nodes:

1. Mesh node queries local bridge node for MESH_ISLAND_ADVERTISE of target island
2. Bridge node queries unified internet DHT if not in local cache
3. Mesh node uses returned island advertisement to find bridge nodes for target island
4. Path construction proceeds

### 14.2 Clearnet Client Discovering Mesh Island Services

A clearnet client with no mesh hardware can discover mesh island services:

1. Client queries unified DHT for MESH_ISLAND_ADVERTISE by island_id
2. Island advertisement lists bridge nodes and services
3. Client constructs cross-domain path via bridge node
4. No special client hardware required

---

## 15. Routing Table Maintenance

### 15.1 Bucket Refresh

Any k-bucket not queried in the last 60 minutes: perform DHT_FIND_NODE for a random key in that bucket's range. This ensures the routing table remains accurate as nodes join and leave.

### 15.2 Dead Node Removal

A node is considered dead if:

- It fails to respond to 3 consecutive DHT queries over a period of 5 minutes

Dead nodes are removed from the routing table and replaced by the next candidate in the replacement cache.

### 15.3 Stale Advertisement Pruning

Storage nodes MUST prune expired advertisements (where `current_time > expires_at`) from their local DHT storage. This frees storage and prevents stale data from being served to clients.

Pool advertisement pruning follows same schedule.

---

## 16. Geographic Plausibility Integration

DHT routing table maintenance includes RTT measurements to known nodes:

1. DHT_FIND_NODE round-trip time measured per query
2. RTT stored per NodeID: `rtt_measurements[node_id] = rolling_average(last_10_rtts)`
3. Discrepancy between advertised region and measured RTT stored as plausibility data
4. Plausibility data fed to reputation system

This measurement happens naturally during normal DHT operation without additional overhead.

Domain-specific RTT thresholds are applied (see `/control-plane/geo_plausibility.md`).

---

## 17. Island Collision Detection

When a mesh island registers island_id in the unified DHT:

1. Perform DHT_GET for island_key before publishing
2. If record exists from different operator: collision detected; choose different island_id
3. island_id should be descriptive and geographically specific (e.g., `lima-san-isidro-2026` not `community-1`)

---

## 18. DHT Privacy Considerations

### 18.1 Query Routing Through Overlay

Once a client has established any BGP-X path, all subsequent DHT queries SHOULD be routed through that path rather than sent directly.

This means:

- The DHT nodes that receive queries see only the exit node's IP, not the client's real IP
- The exit node cannot correlate individual queries with specific clients
- The client's path-building activity is not visible to DHT participants

Domain-filtered queries for mesh island nodes reveal interest in that island to DHT storage nodes.

### 18.2 Query Batching

Clients SHOULD batch DHT queries — fetching multiple node advertisements in a single path traversal — to reduce the number of observable queries.

### 18.3 Prefetching

Clients SHOULD prefetch node advertisements proactively (before they are needed for path construction) and maintain a local cache of valid advertisements.

Cache size: 500–1000 node advertisements
Cache TTL: 30 minutes (independent of advertisement expiry)

Prefetching reduces the correlation between path-construction events and DHT query patterns.

### 18.4 Pool Query Privacy

Pool advertisements are public information — less sensitive than path construction. However, queries for specific pool IDs reveal which pools a client is interested in. Route through established overlay path when possible.

### 18.5 Sybil Resistance in Node Selection

Even if an adversary populates the DHT with many Sybil nodes, they cannot force a client to use those nodes because:

- The path selection algorithm uses weighted random selection (Sybil nodes would need to outcompete legitimate nodes on reputation, uptime, and latency metrics)
- The reputation system flags nodes with anomalous behavior patterns
- Geographic and ASN diversity constraints limit how many nodes a single operator can place in a single path
- Pool curator verification provides additional filter for curated pools

### 18.6 Mesh Island Privacy

Islands that want to remain undiscoverable from clearnet should not publish MESH_ISLAND_ADVERTISE (operate in offline mode only, accessible via known bridge node only).

---

## 19. Network Size Estimation

Nodes SHOULD maintain an estimate of total network size for informational purposes. This is computed using the standard Kademlia method:

```
estimated_network_size = 2^256 / average_distance_between_k_closest_nodes
```

This estimate is used by:
- The reputation system (to detect anomalous concentrations of nodes)
- The path selection algorithm (to estimate anonymity set size)
- Monitoring and metrics

---

## 20. IPv6 Support

BGP-X DHT MUST support IPv6 node addresses.

Nodes with IPv6 addresses SHOULD also advertise an IPv4 address for DHT participation. Clients operating on IPv6-only networks SHOULD use a bootstrap node that supports IPv6.

Extended NodeContact format for IPv6 in current format: use IPv6 address field where applicable.

---

## 21. Summary of Storage Keys

| Record Type | Key Derivation | TTL | Re-publication |
|---|---|---|---|
| Node Advertisement | `BLAKE3("bgpx-node-advert-v1" \|\| node_id)` | 48 hours | 12 hours |
| Pool Advertisement | `BLAKE3("bgpx-pool-v1" \|\| pool_id)` | 7 days | 3 days |
| Domain Bridge | `BLAKE3("bgpx-domain-bridge-v1" \|\| min(domain_a, domain_b) \|\| max(domain_a, domain_b))` | 24 hours | 8 hours |
| Mesh Island | `BLAKE3("bgpx-mesh-island-v1" \|\| island_id)` | 24 hours | 8 hours |
| Pool Key Rotation | `BLAKE3("bgpx-pool-key-rotation-v1" \|\| pool_id \|\| rotation_timestamp)` | accept_until + 7 days | N/A |

---

## 22. DHT Message Types

DHT messages use the BGP-X common header with message types 0x0B through 0x0F, plus extended types 0x1A–0x1C.

| Type | Name | Description |
|---|---|---|
| 0x0B | DHT_FIND_NODE | Locate nodes near a target ID |
| 0x0C | DHT_FIND_NODE_RESP | Response to DHT_FIND_NODE |
| 0x0D | DHT_GET | Retrieve a DHT record |
| 0x0E | DHT_GET_RESP | Response to DHT_GET |
| 0x0F | DHT_PUT | Store a DHT record |
| 0x1A | DOMAIN_ADVERTISE | Announce routing domain endpoints and bridge capability |
| 0x1B | MESH_ISLAND_ADVERTISE | Announce mesh island existence, bridges, and services |
| 0x1C | POOL_KEY_ROTATION | Pool curator key rotation record |

Full wire formats are specified in `/protocol/packet_format.md` and `/protocol/protocol_spec.md`.
