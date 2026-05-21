# BGP-X Gateway Specification

**Version**: 0.1.0-draft
**Last Updated**: 2026-04-24

This document is the unified specification for BGP-X Gateway nodes. It covers the architectural role of gateways, the protocol specification for domain bridge nodes, and the operational guides for Entry and Exit node operators.

---

## 1. Gateway Overview

A BGP-X **Gateway** is a node that bridges traffic between routing domains. The primary role of a gateway is to enable cross-domain connectivity — allowing traffic to flow between the clearnet (BGP-routed internet), the BGP-X overlay, and mesh islands.

Gateway nodes operate in three primary capacities:

1.  **Entry Node**: The first hop in a BGP-X path; accepts client connections and knows the client's transport address.
2.  **Exit Node**: The final hop for clearnet traffic; sends traffic to internet destinations on behalf of BGP-X users.
3.  **Domain Bridge Node**: A node with active endpoints in two or more routing domains (e.g., clearnet + mesh) that forwards traffic between them.

A single node can operate in all three capacities simultaneously. The gateway specification unifies the protocol requirements and operational responsibilities for all three roles.

---

## 2. Domain Bridge Node Specification

A **Domain Bridge Node** is the foundational component that enables BGP-X's cross-domain routing capability. It is any node that advertises endpoints in two or more routing domains.

### 2.1 Bridge Role and Capabilities

The bridge node does **not** decrypt the inner layers of cross-domain traffic. It decrypts only its own `DOMAIN_BRIDGE` hop type layer, determines the target domain, and forwards the remaining encrypted payload using the appropriate transport for the target domain.

**What the Bridge Node Knows:**

| Information | Visible to Bridge Node |
| :--- | :--- |
| Previous hop's transport address | Yes (in current domain) |
| Next hop's transport address | Yes (in target domain) |
| Target routing domain | Yes (from DOMAIN_BRIDGE layer) |
| path_id | Yes (for return routing) |
| Source identity | No |
| Final destination | No |
| Total path composition | No |

### 2.2 Cross-Domain Routing Protocol

When a bridge node receives a packet with `hop_type = DOMAIN_BRIDGE`:

1.  Decrypt the outer onion layer using the session key shared with the client.
2.  Extract `target_domain_id` from the decrypted layer.
3.  Extract `next_hop` address in the target domain.
4.  Retrieve the `path_id` from the layer header.
5.  Store `path_id → predecessor_address` in the **cross-domain path table**.
6.  Forward the remaining encrypted onion payload to `next_hop` via the transport appropriate for `target_domain`.

The bridge node maintains two mappings for return traffic:
*   `path_id → predecessor_in_domain_A`
*   `path_id → predecessor_in_domain_B`

This enables return traffic to traverse the domain boundary without leaking path composition to either side.

### 2.3 MESH_ISLAND_ADVERTISE

Bridge nodes that serve as gateways for mesh islands publish `MESH_ISLAND_ADVERTISE` records to the unified DHT. This record announces the existence of a mesh island, its gateway nodes, and its reachable services.

**Record Structure:**
```json
{
  "island_id": "lima-district-1",
  "domain_id": "0x00000003-a1b2c3d4",
  "gateway_nodes": ["node_id_1", "node_id_2"],
  "services": ["service_id_a", "service_id_b"],
  "bridge_transport": "lora",
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature>"
}
```

### 2.4 DOMAIN_ADVERTISE

Bridge nodes publish `DOMAIN_ADVERTISE` records to signal their capability to bridge specific domain pairs.

**Record Structure:**
```json
{
  "version": 1,
  "node_id": "<NodeID hex>",
  "bridge_pairs": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "bridge_latency_ms": 25,
      "transport": "wifi_mesh",
      "available": true,
      "geo_plausibility_exempt": false
    }
  ],
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "<Ed25519 signature>"
}
```

### 2.5 Unified DHT Participation

Bridge nodes are critical infrastructure for the unified DHT. They store and serve DHT records on behalf of mesh-only nodes that cannot directly access the internet-accessible portion of the DHT.

---

## 3. Entry Node Specification

An entry node is the **first relay hop** in a BGP-X path. It is the **ONLY node** in the network that learns the client's transport address (IP:port for clearnet, NodeID/mesh-address for mesh).

### 3.1 Entry Node Role

The entry node's responsibility is focused on **protecting the client's transport address from forward leakage** rather than protecting destination privacy — destination protection is handled by subsequent relay layers.

**What the Entry Node Knows:**

| Information | Visible to Entry Node |
| :--- | :--- |
| Client's transport address (IP:port) | Yes (from UDP source) |
| Session key (shared with client) | Yes (for this session) |
| Next relay's transport address | Yes (to forward packets) |
| path_id (from HANDSHAKE_DONE) | Yes (stores for return routing) |

**What the Entry Node Does NOT Know:**

| Information | Visible to Entry Node |
| :--- | :--- |
| Destination IP or domain | No |
| Number of total hops in path | No |
| Which routing domains the path traverses | No |
| Whether the path is cross-domain | No |
| Content of the traffic | No (only outer onion layer decrypted) |
| Identity of any hop beyond the next relay | No |
| Whether the path includes mesh, satellite, or overlay segments | No |

### 3.2 Privacy Requirements

Because entry nodes see client transport addresses, they carry a **heavier privacy burden** than relay nodes.

**Entry nodes MUST:**
*   **Never log** client transport addresses (IP, port, or mesh NodeID) in any persistent storage.
*   **Never include** client transport addresses in any metrics, monitoring, or debugging output.
*   **Never pass** client transport addresses to the next relay hop (architecturally prevented by onion encryption).
*   **Zeroize** client transport addresses from memory immediately when sessions close.
*   **Never share** client transport address information with any third party.

**Entry nodes MUST NOT:**
*   Log connection attempts by client transport address.
*   Rate limit connections in a way that creates persistent per-client records.
*   Correlate sessions from the same client transport address across time for analysis.
*   Emit timing or volume statistics that could identify specific clients.

### 3.3 path_id Handling

When a client completes the handshake, the entry node receives `path_id` in the HANDSHAKE_DONE message:

```
entry_node stores: path_id → client_transport_addr (in-memory only, TTL = session idle timeout)
```

This mapping enables **return traffic routing**: when a response arrives for `path_id`, the entry node forwards it back to the client's transport address.

**Critical privacy properties of path_id:**
*   path_id is **in-memory only** — NEVER logged, NEVER persisted to disk.
*   path_id is **random per path** — not linked to client identity.
*   path_id is **unique per path instance** — never reused within a session.
*   path_id is **included in every onion layer** — same value at every hop.
*   path_id **does not leak path composition** — each node sees only path_id, not the full path structure.

### 3.4 Traffic Pattern Protection

Entry nodes SHOULD:
*   Apply **uniform packet processing time** regardless of session content (constant-time where possible).
*   **Batch outbound packets** to reduce timing correlation opportunities.
*   If cover traffic is enabled: generate consistent cover traffic that masks real traffic patterns.

### 3.5 Prohibited Logging

Entry nodes MUST NOT log:
*   Client IP addresses
*   Client NodeIDs (mesh)
*   Client mesh transport addresses
*   Session IDs
*   path_id values
*   Path composition
*   Traffic timing or volume correlated to any client identifier
*   Pool query history
*   Cross-domain traversal details

### 3.6 Configuration

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

### 3.7 Multi-Domain Entry Points

BGP-X provides three equal entry point types. All use the **same handshake protocol** — domain-agnostic.

**Clearnet Entry (UDP/IP)**:
*   Standard operation. Client connects to entry node via UDP over the internet.
*   Entry node sees: client IP:port.
*   Transport: UDP/IP.
*   Handshake: domain-agnostic (identical format).

**Mesh Island Entry**:
*   Client within a mesh island connects to a mesh entry node via radio transport.
*   Entry node sees: client NodeID and mesh transport address (not an IP).
*   Transport: WiFi 802.11s, LoRa, or BLE.
*   Handshake: domain-agnostic (identical format, delivered via mesh radio).

Mesh entry nodes accept connections from mesh clients who have **no ISP IP address**. The privacy implications differ: instead of knowing a client's IP, the entry node knows the client's NodeID and mesh transport address.

**Overlay Entry**:
*   Client already in the BGP-X overlay can extend a path through the overlay domain. The entry node in this case is the first hop of the extension.

### 3.8 Rate Limiting

Entry nodes SHOULD apply rate limiting to new handshake attempts to prevent resource exhaustion and traffic analysis:

```toml
[sessions]
max_new_handshakes_per_source_per_minute = 10
max_sessions = 10000
```

Rate limits apply per source IP (clearnet) or per source NodeID (mesh). Rate limiting MUST NOT create persistent per-client records — use in-memory token buckets with short TTL.

### 3.9 Entry Node Privacy — Additional Mitigations

The entry node's knowledge of the client's transport address is **unavoidable** — the entry node must receive packets from the client. This is the fundamental privacy trade-off of onion routing.

**Client-Side Mitigations**:
*   **Pluggable Transport (PT)**: Obfuscate BGP-X traffic patterns from ISP-level observers.
*   **VPN or proxy in front of BGP-X**: Hide the client's real IP from the entry node (the entry node sees the VPN/proxy IP instead).
*   **Pool-based selection with rotation**: Change entry nodes periodically to avoid long-term observation by any single operator.
*   **Multi-path usage**: Use different entry nodes for different sessions to prevent correlation.

**Entry Node Operator Responsibility**:
Operators MUST:
*   Configure **no-log policy** (mandatory, not optional).
*   Disable any packet capture or network monitoring at the entry node.
*   Ensure no external monitoring (upstream ISP, datacenter logging) captures client IPs.
*   Use full-disk encryption on the node.
*   Run the node in a privacy-friendly jurisdiction if possible.

### 3.10 Pool Participation

Entry nodes can participate in named pools:

```toml
[advertisement]
pool_memberships = ["pool-id-1", "pool-id-2"]
```

Operators register their entry node with pool curators out-of-band. Once accepted, the curator includes the node in the pool advertisement member list.

**Pool Trust Model**:
Pool membership provides a **trust signal** but is not a security guarantee:
*   Pool curators verify node identity and operational standards.
*   Clients can choose pools with curators they trust.
*   Entry nodes in private pools provide operator-controlled trust.
*   Pool membership does NOT guarantee honest behavior — individual node verification always applies.

**Domain-Scoped Pools**:
Pools can be scoped to specific routing domains:
*   **Clearnet-only pools**: Entry nodes with only clearnet endpoints.
*   **Mesh-only pools**: Entry nodes within a specific mesh island.
*   **Domain-bridge pools**: Entry nodes that are also bridge nodes.

Entry nodes advertise their pool memberships in their node advertisement. Clients filter eligible entry nodes by pool during path construction.

### 3.11 Mesh Entry Nodes

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

**Mesh Entry Node Privacy**:
Mesh entry nodes see **client NodeID and mesh transport address**, not an IP address. The privacy implications:
*   **NodeID is pseudonymous**: clients can rotate NodeIDs per session.
*   **Mesh transport address** is a MAC or LoRa device address — local to the island.
*   **No internet-level correlation**: the entry node cannot link mesh clients to their clearnet identities.

### 3.12 Domain Bridge Entry Nodes

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
*   Accepts client connections (entry role).
*   Can route traffic into the mesh island (bridge role).
*   Stores path_id mappings for **both** clearnet-side return routing and cross-domain routing.

**Cross-Domain Session Establishment**:
For cross-domain paths (clearnet client → mesh service):
1.  Client establishes session with bridge/entry node via clearnet (UDP).
2.  Bridge/entry node receives HANDSHAKE_DONE with path_id.
3.  Bridge/entry node stores: `path_id → client_addr` (clearnet-side).
4.  Bridge/entry node forwards handshake to mesh-side relay via its radio.
5.  Bridge/entry node stores: `path_id → mesh_relay_addr` (cross-domain table).
6.  Return traffic from mesh is routed via path_id lookup at bridge, then forwarded to client_addr.

The bridge/entry node **never decrypts** the inner onion layers — it only processes the DOMAIN_BRIDGE hop type at its position in the path.

### 3.13 Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** feature:
*   If an entry node **declares a jurisdiction** in its advertisement: geo plausibility RTT scoring applies.
*   If an entry node **does NOT declare jurisdiction**: geo plausibility does NOT apply.

Entry nodes are NOT required to declare jurisdiction. Declaring jurisdiction:
*   **Pro**: Signals to clients which jurisdiction the node operates in.
*   **Con**: Reveals information about the node's location.

### 3.14 Satellite Internet Entry Nodes

Commercial satellite internet services (Starlink, Iridium, Inmarsat) are **clearnet domain**. A BGP-X entry node with satellite WAN:
*   Accepts client connections via satellite IP.
*   Operates as a clearnet entry node with satellite latency class.
*   Is **exempt** from geo plausibility scoring (satellite IPs may be geographically distant from ground station).

### 3.15 N-Hop Unlimited

Entry nodes enforce **no maximum hop count**. The path length is determined entirely by the client during path construction. The entry node:
*   Sees only the **next hop** (does not know total hops).
*   Cannot enforce a hop limit even if configured to (no protocol field carries total hop count).
*   Processes packets identically regardless of path length.

### 3.16 Availability Requirements

Entry nodes are the **bottleneck for initial connection setup**. Recommended:
*   Uptime: **≥99.0%**.
*   Low latency to target client population.
*   Good connectivity to multiple relay pools.

BGP-X clients select entry nodes using **reputation-weighted random selection**. Low-uptime nodes are selected less frequently. Nodes with poor reputation may be excluded entirely.

### 3.17 Guard Node Mode — REMOVED

**Note**: Earlier BGP-X specification drafts included a "guard node" mode where clients would select a small set of trusted entry nodes for long-term use.

**This concept has been removed from BGP-X.**

Guard nodes create **observable long-term entry relationships** that can be exploited for profiling and timing correlation. The BGP-X architecture now relies on **pool-based trust domains** and **path diversity** instead.

### 3.18 Implementation Requirements

**Entry nodes MUST:**
*   Implement standard relay node functionality.
*   Accept HANDSHAKE_INIT from clients.
*   Store path_id → client_transport_addr mappings in-memory only.
*   Zeroize client transport addresses when sessions close.
*   Never log client transport addresses.
*   Participate in unified DHT.
*   Support domain-agnostic handshake protocol.

**Entry nodes MUST NOT:**
*   Log client transport addresses.
*   Persist path_id mappings to disk.
*   Share client information with any third party.
*   Correlate sessions across time for analysis.
*   Attempt to determine destination from traffic patterns.
*   Enforce a maximum hop count (N-hop unlimited applies).

**Entry nodes SHOULD:**
*   Apply rate limiting to new handshakes.
*   Generate cover traffic if enabled.
*   Maintain high availability.
*   Declare jurisdiction only if willing to accept geo plausibility scoring.

---

## 4. Exit Node Specification

An exit node is the **final hop** for clearnet traffic. The exit node knows certain information but not the full path.

### 4.1 What It Means to Run an Exit Node

Running a BGP-X exit node means that your node sends traffic to clearnet destinations on behalf of BGP-X users. From the perspective of the destination server and any network observer between your node and the destination, the traffic appears to originate from your server's IP address.

This has significant implications:
*   **Legal exposure**: Destinations may receive traffic that violates their terms of service. Law enforcement may associate your IP address with that traffic.
*   **Abuse complaints**: Your IP may appear in abuse reports (spam, scraping, unauthorized access attempts).
*   **Network provider terms**: Your hosting provider may prohibit running exit nodes or forwarding certain traffic.

Running an exit node is a meaningful act of participation in privacy infrastructure. Do not do it without understanding these implications.

### 4.2 Legal Considerations

BGP-X exit nodes are legally analogous to Tor exit nodes and VPN providers. In most jurisdictions, a network intermediary that:
*   Does not select or modify the traffic it carries.
*   Does not know the content it is forwarding.
*   Operates under a published, consistent policy.
*   Responds appropriately to valid legal requests.

...has significant but not unlimited legal protection as a neutral carrier or conduit.

**This is NOT legal advice. Consult a lawyer in your jurisdiction before running an exit node.**

**Jurisdiction Selection**:
Consider running your exit node in a jurisdiction with:
*   Strong network neutrality protections.
*   Clear "safe harbor" or "mere conduit" provisions for ISPs/carriers.
*   A legal tradition of protecting privacy.
*   No mandatory data retention laws (or retention laws that apply only to ISPs, not operators like you).

**Responding to Legal Requests**:
If you receive a legal request (subpoena, court order, law enforcement inquiry) related to your exit node:
1.  Do not panic and do not immediately comply without legal advice.
2.  Consult a lawyer with expertise in internet law in your jurisdiction.
3.  Verify the request is legally valid (proper jurisdiction, proper process).
4.  If you operate a "none" logging policy, explain truthfully that you have no responsive records.
5.  Consider publishing a warrant canary if your jurisdiction permits.

### 4.3 What the Exit Node Sees

An exit node is the final hop for clearnet traffic. The exit node knows certain information but not the full path:

| Information | Visible to Exit Node |
| :--- | :--- |
| Destination IP or domain | Yes |
| Application layer content (if HTTPS) | No (encrypted by TLS) |
| Source (client IP) | No |
| Number of hops before it | No |
| Whether path is cross-domain | No |
| Domain name (if ECH available) | No (hidden by ECH) |
| Domain name (if ECH unavailable) | Yes (from TLS SNI) |

**Important**: Exit nodes do not know whether the path traversed mesh islands before reaching them. A path that traverses multiple mesh islands before reaching the exit node looks identical to the exit node as a single-domain clearnet path.

### 4.4 Exit Policy Enforcement

All exit node operators MUST publish a signed exit policy. The exit policy is enforced technically, not by trust.

When a connection is denied by exit policy: `STREAM_CLOSE` with `ERR_EXIT_POLICY_DENIED`. The exit node does NOT reveal why the connection was denied or what the deny list contains.

**Minimal Risk Exit Policy**:
This policy minimizes abuse potential while still providing useful exit capability:

```toml
[exit_policy]
allow_protocols = ["tcp"]
allow_ports = [80, 443]  # HTTP and HTTPS only
deny_destinations = [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
    "100.64.0.0/10",    # Shared address space
    "192.0.0.0/24",     # IETF protocol assignments
    "198.51.100.0/24",  # TEST-NET-2
    "203.0.113.0/24",   # TEST-NET-3
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "DE"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

**Full-Service Exit Policy**:
For operators willing to accept more risk in exchange for more utility:

```toml
[exit_policy]
allow_protocols = ["tcp", "udp"]
allow_ports = []  # All ports
deny_destinations = [
    # RFC 1918 and other non-routable
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "169.254.0.0/16",
    "::1/128",
    "fc00::/7",
    "fe80::/10",
]
logging_policy = "none"
operator_contact = "exit-ops@yourdomain.com"
jurisdiction = "IS"  # Iceland
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

### 4.5 DNS Resolution

All DNS resolution at exit nodes MUST use DoH (DNS over HTTPS) with DNSSEC validation:

```toml
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
```

ECS (EDNS Client Subnet) MUST be stripped from all DNS queries. Revealing the client subnet to DNS resolvers would leak approximate client location.

DoH is required because:
*   Plain DNS (UDP/53) is visible to network observers.
*   DoT (DNS over TLS) is acceptable but DoH is preferred for its HTTPS disguise.
*   The exit node's DNS queries should not reveal BGP-X traffic patterns.

### 4.6 ECH (Encrypted Client Hello)

When `ech_capable = true` and the destination supports ECH:

1.  Exit node queries DNS for destination's ECH configuration (HTTPS DNS record).
2.  Constructs TLS ClientHello with ECH (using destination's ECH public key).
3.  TLS ServerName Indication (SNI) contains a cover name, not the real domain.
4.  Real destination domain is encrypted inside ECH and only decryptable by the destination server.

ECH hides the domain name from network-level observers between the exit node and destination. This includes the exit node's own ISP and any passive monitoring.

**ECH does not apply if destination does not publish ECH configuration.**

**ECH Configuration and Testing**:
Verify ECH is working:
```bash
bgpx-cli exit test-ech --destination example.com
```

**ECH Caching**:
Exit nodes cache ECH configurations with a 1-hour TTL.

### 4.7 Exit Policy Versioning

When exit policy changes:
1.  Increment `exit_policy_version`.
2.  Set `exit_policy_previous_version` to old version number.
3.  Re-publish advertisement immediately.
4.  Old policy honored for existing streams until they close.
5.  New policy applied to new streams immediately.

### 4.8 Abuse Handling

**Expected Abuse Categories**:

| Category | Frequency | Severity | Response |
| :--- | :--- | :--- | :--- |
| DMCA/copyright notices | Common | Low | Forward notice; no logs available |
| Port scanning complaints | Occasional | Low | Explain exit node nature |
| Spam complaints | Rare (if port 25 blocked) | Medium | Verify port 25 is blocked |
| Hack attempt reports | Rare | Medium | Explain; review exit policy |
| Law enforcement inquiry | Rare | High | Consult lawyer immediately |

**Abuse Contact Template**:
Publish the following (or similar) at your abuse contact address:
```
This IP address is a BGP-X exit node.
BGP-X is a privacy-preserving overlay routing network similar to Tor.
Traffic originating from this IP address was forwarded on behalf of
BGP-X users. We do not log, store, or have access to information
about the source of this traffic or its content.
```

### 4.9 Monitoring

**Key Metrics to Monitor**:

| Metric | Alert threshold | Description |
| :--- | :--- | :--- |
| Active outbound connections | > 1,800 (90% of limit) | Approaching connection limit |
| Exit policy denial rate | > 20% of streams | Possible misconfiguration |
| DNS resolution failures | > 5% of resolutions | DNS resolver issue |
| Bandwidth utilization | > 90% of cap | Approaching bandwidth limit |
| Session count | > 9,000 (90% of limit) | Approaching session limit |
| ECH fallback rate | High (informational) | Many destinations without ECH |

**Health Check Endpoint**:
```bash
bgpx-cli stats
```

**IP Reputation Monitoring**:
Monitor your exit node's IP address in public IP reputation services:
*   AbuseIPDB
*   Spamhaus
*   Barracuda Reputation Block List

### 4.10 Pool Participation for Exit Nodes

Register with curated exit pools to increase visibility to privacy-conscious users.

Pool membership signals trust attributes to clients:
*   **No-log policy**: Verified by pool curator.
*   **ECH capable**: Destination domain privacy.
*   **Jurisdiction**: Legal environment transparency.
*   **Uptime**: Reliability commitment.

### 4.11 Mesh Gateway Operations

If also operating as a gateway for a mesh network:
```bash
bgpx-cli gateway status
bgpx-cli gateway dht-sync-status
bgpx-cli mesh peers
```

**Gateway Reliability**:
Gateway uptime is more critical than relay uptime — mesh users depend on the gateway for clearnet access. Consider:
*   Uninterruptible power supply (UPS).
*   Redundant WAN connections (fiber + cellular).
*   Automatic failover between WAN links.
*   Physical security for the gateway node.

**Cross-Domain Path Handling**:
As a mesh gateway (domain bridge node), your gateway:
*   Bridges between mesh routing domain and clearnet domain.
*   Stores path_id entries in both single-domain and cross-domain path tables.
*   Forwards traffic between domains without re-encryption.
*   Does NOT know the client's mesh address when handling return traffic (path_id routing).

### 4.12 Satellite WAN Considerations

If your exit node uses satellite internet (Starlink, Iridium, Inmarsat, HughesNet, Viasat) as its WAN connection:
*   **Domain**: Still clearnet (0x00000001), not a separate satellite domain.
*   **Latency class**: Configured as `satellite-leo` (20-60ms) or `satellite-geo` (600ms+).
*   **Geo plausibility**: Satellite nodes are exempt from geographic plausibility RTT scoring.

### 4.13 Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal for exit nodes:
*   **If jurisdiction is declared**: geo plausibility RTT scoring applies.
*   **If jurisdiction is NOT declared**: geo plausibility scoring does NOT apply.

Satellite-connected exit nodes are **exempt** from geo plausibility scoring regardless of jurisdiction declaration.

### 4.14 Operator Responsibilities

Exit node operators are responsible for:
*   Publishing accurate exit policy.
*   Responding to abuse complaints via `operator_contact`.
*   Complying with local laws regarding traffic they handle.
*   NOT logging connection-level data (technical enforcement by `logging_policy = "none"`).
*   Running up-to-date BGP-X software.
*   Maintaining adequate uptime (especially for gateways serving mesh communities).
*   Understanding cross-domain implications if operating as a domain bridge.

**BGP-X provides no legal protection.** Operators should consult legal advice for their jurisdiction.

---

## 5. Hardware Recommendations

| Hardware | Use Case | Notes |
| :--- | :--- | :--- |
| BGP-X Router v1 (Outdoor) | Small community gateway | Full daemon, integrated radios, IP67 |
| BGP-X Node v1 | Community mesh gateway | Solar-powered option, outdoor enclosure |
| BGP-X Gateway v1 | Provider exit infrastructure | High throughput, SFP+ uplink, 1 GB RAM |
| GL.iNet GL-MT6000 + LoRa USB | Low-cost domain bridge | Third-party hardware, requires adapter |
| x86 server | Datacenter exit node | Maximum performance, high session capacity |

---

## 6. Checklist for New Gateway Operators

Before announcing your gateway node:

- [ ] Understand legal implications for your jurisdiction.
- [ ] Configure minimal-risk or appropriate exit policy.
- [ ] Enable DoH DNS with DNSSEC validation.
- [ ] Enable ECH capability (`ech_capable = true`).
- [ ] Set `logging_policy = "none"`.
- [ ] Configure `operator_contact` and `jurisdiction` (optional).
- [ ] Test exit functionality with known destinations.
- [ ] Test ECH functionality.
- [ ] Verify no prohibited data is logged.
- [ ] Set up monitoring and health checks.
- [ ] Prepare abuse response template.
- [ ] Join relevant exit pools.
- [ ] If operating as mesh gateway: configure domain bridge settings.

---

## 7. References

- `/protocol/protocol_spec.md` — Exit policy wire format
- `/protocol/handshake.md` — Domain-agnostic handshake specification
- `/protocol/path_construction.md` — Path building including entry node selection
- `/control-plane/reputation_system.md` — Reputation scoring and node selection
- `/control-plane/geo_plausibility.md` — Geographic plausibility scoring (optional)
- `/protocol/routing_domains.md` — Domain types and cross-domain routing
- `/hardware/router_spec.md` — BGP-X Router v1 hardware
- `/hardware/gateway_spec.md` — BGP-X Gateway v1 hardware
- `/docs/satellite_architecture.md` — Satellite WAN integration
