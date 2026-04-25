# BGP-X Node Discovery Specification

**Version**: 0.1.0-draft

This document specifies how BGP-X nodes discover each other and how clients find available relay nodes to use in path construction.

---

## 1. Overview

BGP-X uses a **Distributed Hash Table (DHT)** for node discovery. The DHT is the mechanism by which nodes publish their existence and capabilities, and by which clients find nodes to use in paths.

The BGP-X DHT has the following design properties:

- **Fully decentralized**: No central server, no directory authority, no single point of failure or coercion
- **Self-organizing**: Nodes join and leave without coordination
- **Cryptographically authenticated**: All records are signed; unsigned or invalid records are rejected
- **Privacy-preserving**: DHT queries for nodes are distinguishable from path-construction traffic only by an adversary who can observe both the query and the response at the network level
- **Sybil-resistant**: The cost of populating the DHT with fake nodes is bounded by the signature verification requirement and the reputation system

---

## 2. DHT Algorithm

BGP-X uses a **Kademlia**-style DHT. Kademlia is well-studied, widely deployed (used in BitTorrent, IPFS, Ethereum), and provides efficient O(log N) lookup in networks of N nodes.

### 2.1 Key space

The DHT key space is 256 bits, matching the output length of BLAKE3.

All DHT keys are 256-bit values. Node positions in the DHT are determined by their NodeID:

```
node_position = NodeID = BLAKE3(Ed25519_public_key)
```

### 2.2 Distance metric

Kademlia uses XOR as a distance metric:

```
distance(a, b) = a XOR b
```

This metric is symmetric and satisfies the triangle inequality, making it suitable for routing decisions.

### 2.3 Routing table (k-bucket structure)

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

### 2.4 Lookup algorithm

To find nodes near a target key:

1. Select the α closest known nodes to the target (BGP-X default: α = 3)
2. Send parallel DHT_FIND_NODE queries to all α nodes
3. From the responses, identify the k closest nodes found so far
4. Send queries to any newly discovered nodes that are closer than the current k-closest
5. Repeat until no closer nodes are discovered
6. Return the k closest nodes found

BGP-X uses α = 3 (3 parallel queries) as the default concurrency parameter. Higher α values speed up lookups at the cost of more network traffic.

---

## 3. Node Advertisement Storage

Node advertisements are stored in the DHT at the key:

```
storage_key = BLAKE3("bgpx-node-advert-v1" || node_id)
```

The prefix namespaces BGP-X records from other DHT record types and from future BGP-X record types.

### 3.1 Storing an advertisement

A node publishes its advertisement by:

1. Constructing and signing the advertisement (see `/control-plane/node_advertisement.md`)
2. Computing the storage key
3. Finding the k nodes closest to the storage key via DHT lookup
4. Sending DHT_PUT to each of those k nodes

The k closest nodes to the storage key are responsible for storing the advertisement and serving it to requesters.

### 3.2 Retrieving an advertisement

A client retrieves a node's advertisement by:

1. Computing the storage key from the known NodeID
2. Finding the k nodes closest to the storage key
3. Sending DHT_GET to those nodes
4. Receiving the signed advertisement record
5. Verifying the signature and expiry before using the record

### 3.3 Re-publication

Node advertisements expire after a maximum of 48 hours (set in the `expires_at` field). Nodes MUST re-publish their advertisement before it expires.

Recommended re-publication interval: **every 12 hours**

When re-publishing, nodes MUST update the `signed_at` and `expires_at` timestamps and re-sign the advertisement.

Nodes that fail to re-publish within 48 hours will disappear from the DHT naturally as their records expire on storage nodes.

---

## 4. Bootstrap Process

A new node joining the BGP-X network has no routing table. It uses bootstrap nodes to gain initial DHT access.

### 4.1 Bootstrap nodes

Bootstrap nodes are well-known BGP-X nodes whose IP addresses and NodeIDs are hardcoded in the BGP-X software distribution.

Properties of bootstrap nodes:

- Run by the BGP-X project or trusted community operators
- High uptime requirement (99%+)
- Geographically distributed across multiple continents and jurisdictions
- Do NOT serve as relay or exit nodes (they serve DHT only)
- Their NodeIDs and public keys are published in the software and independently verifiable

Bootstrap nodes are not trusted authorities. They are entry points into the DHT graph. Once a joining node has populated its routing table, bootstrap nodes are no longer needed for routing.

### 4.2 Bootstrap sequence

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

### 4.3 Bootstrap node failure tolerance

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

## 5. DHT Privacy Considerations

The DHT presents privacy challenges for clients:

- DHT queries reveal the NodeIDs the client is interested in
- A DHT node that receives many queries from the same IP can infer that IP is building paths
- A DHT node could serve manipulated advertisements (Sybil nodes) to influence path selection

### 5.1 Query routing through overlay

Once a client has established any BGP-X path, all subsequent DHT queries SHOULD be routed through that path rather than sent directly.

This means:

- The DHT nodes that receive queries see only the exit node's IP, not the client's real IP
- The exit node cannot correlate individual queries with specific clients
- The client's path-building activity is not visible to DHT participants

### 5.2 Query batching

Clients SHOULD batch DHT queries — fetching multiple node advertisements in a single path traversal — to reduce the number of observable queries.

### 5.3 Prefetching

Clients SHOULD prefetch node advertisements proactively (before they are needed for path construction) and maintain a local cache of valid advertisements.

Cache size: 500–1000 node advertisements
Cache TTL: 30 minutes (independent of advertisement expiry)

Prefetching reduces the correlation between path-construction events and DHT query patterns.

### 5.4 Sybil resistance in node selection

Even if an adversary populates the DHT with many Sybil nodes, they cannot force a client to use those nodes because:

- The path selection algorithm uses weighted random selection (Sybil nodes would need to outcompete legitimate nodes on reputation, uptime, and latency metrics)
- The reputation system flags nodes with anomalous behavior patterns
- Geographic and ASN diversity constraints limit how many nodes a single operator can place in a single path

---

## 6. DHT Maintenance

### 6.1 Routing table refresh

Nodes MUST refresh any k-bucket that has not been queried in the last 60 minutes by performing a DHT_FIND_NODE for a random key in that bucket's range. This ensures the routing table remains accurate as nodes join and leave.

### 6.2 Dead node removal

A node is considered dead if:

- It fails to respond to 3 consecutive DHT queries over a period of 5 minutes

Dead nodes are removed from the routing table and replaced by the next candidate in the replacement cache.

### 6.3 Stale advertisement pruning

Storage nodes MUST prune expired advertisements (where current time > `expires_at`) from their local DHT storage. This frees storage and prevents stale data from being served to clients.

---

## 7. DHT Message Formats

DHT messages use the BGP-X common header with message types 0x0B through 0x0F. Full wire formats are specified in `/protocol/packet_format.md` and `/protocol/protocol_spec.md`.

### 7.1 DHT_FIND_NODE (0x0B)

Sent to discover nodes near a target NodeID.

Request payload:
- Target NodeID (32 bytes)
- Max results (1 byte, 1–20, default 20)

Response (DHT_FIND_NODE_RESP, 0x0C):
- Result count (1 byte)
- Array of NodeContact entries (40 bytes each)

### 7.2 DHT_PUT (0x0F)

Sent to store a node advertisement.

Payload:
- Record length (4 bytes)
- Signed node advertisement JSON (variable, max 4096 bytes)

Response: None (fire-and-forget). Failures are handled by redundant PUT to multiple storage nodes.

### 7.3 DHT_GET (0x0D)

Sent to retrieve a node advertisement by NodeID.

Request payload:
- Target NodeID (32 bytes)

Response (DHT_GET_RESP, 0x0E):
- Found flag (1 byte)
- Record length (4 bytes, if found)
- Signed node advertisement JSON (variable, if found)

---

## 8. Network Size Estimation

Nodes SHOULD maintain an estimate of total network size for informational purposes. This is computed using the standard Kademlia method:

```
estimated_network_size = 2^256 / average_distance_between_k_closest_nodes
```

This estimate is used by:
- The reputation system (to detect anomalous concentrations of nodes)
- The path selection algorithm (to estimate anonymity set size)
- Monitoring and metrics

---

## 9. IPv6 Support

BGP-X DHT MUST support IPv6 node addresses. NodeContact entries carry IPv4 addresses in the current format. An extended NodeContact format for IPv6 will be introduced in a future MINOR version.

For the current version, nodes with IPv6 addresses SHOULD also advertise an IPv4 address for DHT participation. Clients operating on IPv6-only networks SHOULD use a bootstrap node that supports IPv6.
