# BGP-X Entry Node Specification

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

---

## 1. Entry Node Role

An entry node is the **first relay hop** in a BGP-X path. It is the **ONLY node** in the network that learns the client's transport address (IP:port for clearnet, NodeID/mesh-address for mesh).

The entry node's responsibility is focused on **protecting the client's transport address from forward leakage** rather than protecting destination privacy — destination protection is handled by subsequent relay layers.

### 1.1 What the Entry Node Knows

| Information | Visible to Entry Node |
|---|---|
| Client's transport address (IP:port) | ✅ Yes (from UDP source) |
| Session key (shared with client) | ✅ Yes (for this session) |
| Next relay's transport address | ✅ Yes (to forward packets) |
| path_id (from HANDSHAKE_DONE) | ✅ Yes (stores for return routing) |

### 1.2 What the Entry Node Does NOT Know

| Information | Visible to Entry Node |
|---|---|
| Destination IP or domain | ❌ No |
| Number of total hops in path | ❌ No |
| Which routing domains the path traverses | ❌ No |
| Whether the path is cross-domain | ❌ No |
| Content of the traffic | ❌ No (only outer onion layer decrypted) |
| Identity of any hop beyond the next relay | ❌ No |
| Whether the path includes mesh, satellite, or overlay segments | ❌ No |

The entry node decrypts only its own onion layer. All information about subsequent hops, destination, domain crossings, and path composition remains encrypted inside the inner layers.

---

## 2. Privacy Requirements

Because entry nodes see client transport addresses, they carry a **heavier privacy burden** than relay nodes.

### 2.1 Client Transport Address Handling

Entry nodes MUST:

- **Never log** client transport addresses (IP, port, or mesh NodeID) in any persistent storage
- **Never include** client transport addresses in any metrics, monitoring, or debugging output
- **Never pass** client transport addresses to the next relay hop (architecturally prevented by onion encryption)
- **Zeroize** client transport addresses from memory immediately when sessions close
- **Never share** client transport address information with any third party

Entry nodes MUST NOT:

- Log connection attempts by client transport address
- Rate limit connections in a way that creates persistent per-client records
- Correlate sessions from the same client transport address across time for analysis
- Emit timing or volume statistics that could identify specific clients

### 2.2 path_id Handling

When a client completes the handshake, the entry node receives `path_id` in the HANDSHAKE_DONE message:

```
entry_node stores: path_id → client_transport_addr (in-memory only, TTL = session idle timeout)
```

This mapping enables **return traffic routing**: when a response arrives for `path_id`, the entry node forwards it back to the client's transport address.

**Critical privacy properties of path_id**:

- path_id is **in-memory only** — NEVER logged, NEVER persisted to disk
- path_id is **random per path** — not linked to client identity
- path_id is **unique per path instance** — never reused within a session
- path_id is **included in every onion layer** — same value at every hop
- path_id **does not leak path composition** — each node sees only path_id, not the full path structure

For cross-domain paths, path_id routing extends across domain boundaries. The domain bridge node stores cross-domain path_id mappings to enable return traffic across the clearnet/mesh boundary. The entry node (in clearnet domain) never learns that the path crosses into mesh.

### 2.3 Traffic Pattern Protection

Entry nodes SHOULD:

- Apply **uniform packet processing time** regardless of session content (constant-time where possible)
- **Batch outbound packets** to reduce timing correlation opportunities
- If cover traffic is enabled: generate consistent cover traffic that masks real traffic patterns

### 2.4 Prohibited Logging

Entry nodes MUST NOT log:

- Client IP addresses
- Client NodeIDs (mesh)
- Client mesh transport addresses
- Session IDs
- path_id values
- Path composition
- Traffic timing or volume correlated to any client identifier
- Pool query history
- Cross-domain traversal details

---

## 3. Entry Node Configuration

Any relay node can accept entry node connections. The `entry` role is not exclusive — nodes can be relay, entry, and discovery simultaneously.

```toml
[node]
roles = ["relay", "entry", "discovery"]

[sessions]
max_sessions = 10000        # Entry nodes see more sessions than relay-only
idle_timeout_secs = 90       # Standard session timeout

[network]
listen_addr = "0.0.0.0"
listen_port = 7474
```

Entry nodes must have at least one **publicly reachable endpoint** in their `routing_domains` for clients to discover and connect.

---

## 4. Multi-Domain Entry Points

BGP-X provides three equal entry point types. All use the **same handshake protocol** — domain-agnostic.

### 4.1 Clearnet Entry (UDP/IP)

Standard operation. Client connects to entry node via UDP over the internet.

- Entry node sees: client IP:port
- Transport: UDP/IP
- Handshake: domain-agnostic (identical format)

### 4.2 Mesh Island Entry

Client within a mesh island connects to a mesh entry node via radio transport.

- Entry node sees: client NodeID and mesh transport address (not an IP)
- Transport: WiFi 802.11s, LoRa, or BLE
- Handshake: domain-agnostic (identical format, delivered via mesh radio)

Mesh entry nodes accept connections from mesh clients who have **no ISP IP address**. The privacy implications differ: instead of knowing a client's IP, the entry node knows the client's NodeID and mesh transport address.

### 4.3 Overlay Entry

Client already in the BGP-X overlay can extend a path through the overlay domain. The entry node in this case is the first hop of the extension.

All three entry types use the same `HANDSHAKE_INIT → HANDSHAKE_RESP → HANDSHAKE_DONE` sequence. The entry node does not know which domain the client originates from.

---

## 5. Rate Limiting

Entry nodes SHOULD apply rate limiting to new handshake attempts to prevent resource exhaustion and traffic analysis:

```toml
[sessions]
max_new_handshakes_per_source_per_minute = 10
max_sessions = 10000
```

Rate limits apply per source IP (clearnet) or per source NodeID (mesh). Rate limiting MUST NOT create persistent per-client records — use in-memory token buckets with short TTL.

---

## 6. Entry Node Privacy — Additional Mitigations

The entry node's knowledge of the client's transport address is **unavoidable** — the entry node must receive packets from the client. This is the fundamental privacy trade-off of onion routing.

### 6.1 Client-Side Mitigations

Clients can take additional steps to protect their transport address:

- **Pluggable Transport (PT)**: Obfuscate BGP-X traffic patterns from ISP-level observers
- **VPN or proxy in front of BGP-X**: Hide the client's real IP from the entry node (the entry node sees the VPN/proxy IP instead)
- **Pool-based selection with rotation**: Change entry nodes periodically to avoid long-term observation by any single operator
- **Multi-path usage**: Use different entry nodes for different sessions to prevent correlation

### 6.2 Entry Node Operator Responsibility

Entry node operators MUST:

- Configure **no-log policy** (mandatory, not optional)
- Disable any packet capture or network monitoring at the entry node
- Ensure no external monitoring (upstream ISP, datacenter logging) captures client IPs
- Use full-disk encryption on the node
- Run the node in a privacy-friendly jurisdiction if possible

---

## 7. Pool Participation

Entry nodes can participate in named pools:

```toml
[advertisement]
pool_memberships = ["pool-id-1", "pool-id-2"]
```

Operators register their entry node with pool curators out-of-band. Once accepted, the curator includes the node in the pool advertisement member list.

### 7.1 Pool Trust Model

Pool membership provides a **trust signal** but is not a security guarantee:

- Pool curators verify node identity and operational standards
- Clients can choose pools with curators they trust
- Entry nodes in private pools provide operator-controlled trust
- Pool membership does NOT guarantee honest behavior — individual node verification always applies

### 7.2 Domain-Scoped Pools

Pools can be scoped to specific routing domains:

- **Clearnet-only pools**: Entry nodes with only clearnet endpoints
- **Mesh-only pools**: Entry nodes within a specific mesh island
- **Domain-bridge pools**: Entry nodes that are also bridge nodes

Entry nodes advertise their pool memberships in their node advertisement. Clients filter eligible entry nodes by pool during path construction.

---

## 8. Mesh Entry Nodes

In mesh deployments, entry nodes accept connections from mesh transport:

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "lima-district-1"
transports = ["wifi_mesh", "lora"]

[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ protocol = "udp", address = "203.0.113.1", port = 7474 }]
```

A mesh entry node can also be a **domain bridge node** if it has both mesh and clearnet endpoints.

### 8.1 Mesh Entry Node Privacy

Mesh entry nodes see **client NodeID and mesh transport address**, not an IP address. The privacy implications:

- **NodeID is pseudonymous**: clients can rotate NodeIDs per session
- **Mesh transport address** is a MAC or LoRa device address — local to the island
- **No internet-level correlation**: the entry node cannot link mesh clients to their clearnet identities

---

## 9. Domain Bridge Entry Nodes

A node can serve as both an entry node and a domain bridge node simultaneously:

```toml
[node]
roles = ["relay", "entry", "domain_bridge"]

[advertisement]
bridge_capable = true
bridges = [
    {
        "from_domain": "0x00000001-00000000",
        "to_domain": "0x00000003-a1b2c3d4",
        "available": true
    }
]
```

A domain bridge entry node:

- Accepts client connections (entry role)
- Can route traffic into the mesh island (bridge role)
- Stores path_id mappings for **both** clearnet-side return routing and cross-domain routing

### 9.1 Cross-Domain Session Establishment

For cross-domain paths (clearnet client → mesh service):

1. Client establishes session with bridge/entry node via clearnet (UDP)
2. Bridge/entry node receives HANDSHAKE_DONE with path_id
3. Bridge/entry node stores: `path_id → client_addr` (clearnet-side)
4. Bridge/entry node forwards handshake to mesh-side relay via its radio
5. Bridge/entry node stores: `path_id → mesh_relay_addr` (cross-domain table)
6. Return traffic from mesh is routed via path_id lookup at bridge, then forwarded to client_addr

The bridge/entry node **never decrypts** the inner onion layers — it only processes the DOMAIN_BRIDGE hop type at its position in the path.

---

## 10. Unified DHT Participation

Entry nodes participate in the **unified DHT**:

- Store and serve node advertisements
- Participate in DHT queries (find_node, get, put)
- Cache mesh island records when acting as a bridge/gateway

Entry nodes do NOT maintain a separate DHT. There is one unified DHT spanning all routing domains.

---

## 11. Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** feature:

- If an entry node **declares a jurisdiction** in its advertisement: geo plausibility RTT scoring applies
- If an entry node **does NOT declare jurisdiction**: geo plausibility does NOT apply

Entry nodes are NOT required to declare jurisdiction. Declaring jurisdiction:

- **Pro**: Signals to clients which jurisdiction the node operates in
- **Con**: Reveals information about the node's location

For entry nodes, jurisdiction declaration has additional privacy considerations because the node sees client IPs. Clients may prefer entry nodes in specific jurisdictions for legal protection, or may prefer nodes in **different** jurisdictions from themselves.

---

## 12. Satellite Internet Entry Nodes

Commercial satellite internet services (Starlink, Iridium, Inmarsat) are **clearnet domain**. A BGP-X entry node with satellite WAN:

- Accepts client connections via satellite IP
- Operates as a clearnet entry node with satellite latency class
- Is **exempt** from geo plausibility scoring (satellite IPs may be geographically distant from ground station)

Satellite entry nodes should expect:
- Higher KEEPALIVE intervals (extended for satellite RTT)
- Clients may experience 20-600ms additional latency
- Bandwidth limits vary by satellite provider

---

## 13. HTTP/2 and Native Services

Entry nodes handling traffic to BGP-X native services (.bgpx):

- Traffic is carried over BGP-X streams
- Native services use **HTTP/2 over BGP-X streams** (not HTTP/3)
- HTTP/3 is used only at exit nodes for clearnet HTTP/3 servers

Entry nodes do not interpret HTTP — they forward encrypted onion payloads to subsequent relays.

---

## 14. N-Hop Unlimited

Entry nodes enforce **no maximum hop count**. The path length is determined entirely by the client during path construction. The entry node:

- Sees only the **next hop** (does not know total hops)
- Cannot enforce a hop limit even if configured to (no protocol field carries total hop count)
- Processes packets identically regardless of path length

---

## 15. Availability Requirements

Entry nodes are the **bottleneck for initial connection setup**. Recommended:

- Uptime: **≥99.0%**
- Low latency to target client population
- Good connectivity to multiple relay pools

BGP-X clients select entry nodes using **reputation-weighted random selection**. Low-uptime nodes are selected less frequently. Nodes with poor reputation may be excluded entirely.

---

## 16. Entry vs. Relay Trade-offs

| Consideration | Entry Role | Relay Only |
|---|---|---|
| Client transport address exposure | Yes — sees IP or NodeID | No |
| Legal exposure | Slightly higher (knows who connects) | Lower |
| Client-facing latency impact | Direct | Indirect |
| Privacy responsibility | Higher (no-log mandatory) | Same (no-log) |
| Configuration complexity | Same daemon, additional roles | Simpler |
| DHT participation | Yes | Yes |
| Can be domain bridge | Yes | Yes |
| Mesh-capable | Yes (with mesh transport) | Yes (with mesh transport) |

---

## 17. Recommendations for New Operators

1. **Start with relay-only** before adding entry role
2. Understand the privacy responsibilities before accepting client connections
3. Configure strict no-log policy
4. Test in a clearnet-only environment before enabling mesh transport
5. If operating a mesh entry node: ensure clear understanding of mesh addressing (NodeID vs IP)
6. If operating as a domain bridge entry node: understand cross-domain path_id routing

---

## 18. Guard Node Mode — REMOVED

**Note**: Earlier BGP-X specification drafts included a "guard node" mode where clients would select a small set of trusted entry nodes for long-term use.

**This concept has been removed from BGP-X.**

Guard nodes create **observable long-term entry relationships** that can be exploited for profiling and timing correlation. The BGP-X architecture now relies on **pool-based trust domains** and **path diversity** instead:

- Clients select from pools of nodes rather than pinning to specific entries
- Pool diversity and operator diversity prevent any single operator from observing all traffic
- Path rotation (different entry nodes per session) is handled by the path construction algorithm
- No long-term "guard" relationship is established or needed

If a client wants to use the same entry node repeatedly for operational reasons, they can configure a **private pool** containing only that node. This provides the operational benefit without implying a protocol-level guard mechanism.

---

## 19. Implementation Requirements

Entry nodes MUST:

- Implement standard relay node functionality
- Accept HANDSHAKE_INIT from clients
- Store path_id → client_transport_addr mappings in-memory only
- Zeroize client transport addresses when sessions close
- Never log client transport addresses
- Participate in unified DHT
- Support domain-agnostic handshake protocol

Entry nodes MUST NOT:

- Log client transport addresses
- Persist path_id mappings to disk
- Share client information with any third party
- Correlate sessions across time for analysis
- Attempt to determine destination from traffic patterns
- Enforce a maximum hop count (N-hop unlimited applies)

Entry nodes SHOULD:

- Apply rate limiting to new handshakes
- Generate cover traffic if enabled
- Maintain high availability
- Declare jurisdiction only if willing to accept geo plausibility scoring

---

## 20. Related Documentation

- `/protocol/handshake.md` — Domain-agnostic handshake specification
- `/protocol/path_construction.md` — Path building including entry node selection
- `/control-plane/reputation_system.md` — Reputation scoring and node selection
- `/control-plane/geo_plausibility.md` — Geographic plausibility scoring (optional)
- `/protocol/routing_domains.md` — Domain types and cross-domain routing
- `/hardware/hardware_spec.md` — Hardware requirements for entry nodes
