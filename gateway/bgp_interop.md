# BGP-X and BGP Interoperability Specification

**Version**: 0.1.0-draft

This document specifies the precise relationship between BGP (Border Gateway Protocol) and BGP-X, including gateway behavior, the BGP replacement spectrum, coverage gap bridging, and technical boundary conditions.

---

## 1. Core Relationship

BGP-X does not participate in BGP. BGP-X does not modify BGP. BGP-X does not require BGP operators to make any changes. BGP-X uses the public internet — which is routed by BGP — as one transport medium among several.

They operate at different layers with no direct coupling.

```
BGP-X Application Layer          ← Applications, SDK
BGP-X Domain Router              ← Cross-domain path construction
BGP-X Overlay Routing            ← Path selection, onion encryption, unified DHT
BGP-X Transport (UDP/IP)         ← BGP-X packets as UDP datagrams on clearnet
BGP-X Transport (Mesh/Satellite) ← BGP-X on radio or satellite transport
         │
         │  Clearnet UDP datagrams are ordinary UDP/7474 to BGP routers
         ▼
BGP Routing Layer                ← ISP routing, AS-level decisions
Physical / Link Layer            ← fiber, radio, physical infrastructure
```

For mesh deployments, the BGP-X Transport layer is replaced by radio (LoRa, WiFi mesh, BLE) with zero BGP involvement. A mesh island's internal traffic is completely invisible to BGP.

This relationship is one-directional in terms of visibility:

- BGP can "see" BGP-X packets (as UDP datagrams with opaque payloads)
- BGP-X can "see" BGP in the sense that it uses BGP-routed paths between nodes
- BGP cannot see inside BGP-X packets
- BGP-X cannot modify BGP routing decisions

See `docs/bgp_bgpx_coexistence.md` for the complete architectural explanation.

---

## 2. What BGP Sees From BGP-X Traffic

To every BGP router along the clearnet path, BGP-X packets appear as:

```
Source IP:      Client/router WAN IP
Destination IP: BGP-X relay node IP
Protocol:       UDP
Port:           7474 (default)
Payload:        Encrypted bytes (opaque)
```

BGP cannot see inside. BGP cannot determine these are BGP-X packets. BGP cannot see the actual destination or whether a path is cross-domain.

**For cross-domain paths**: when clearnet segments of a cross-domain path traverse the internet, BGP sees exactly this — ordinary UDP between server IPs. BGP cannot determine the traffic is destined for a mesh island or that the sender is a mesh community member.

---

## 3. What BGP-X Sees From BGP

BGP-X treats the public internet as a best-effort packet delivery service. BGP-X:

- Knows relay node IPs from signed DHT advertisements
- Sends UDP to those IPs
- Has NO visibility into BGP tables, AS paths, or BGP decisions
- Does NOT participate in BGP peering
- Does NOT announce routes
- Does NOT require ISP cooperation
- Uses ASN information only for path diversity enforcement (self-reported by node operators in their advertisements)

BGP-X does use ASN information — but only for path diversity enforcement. It reads the ASN from node advertisements and uses it to ensure no two hops in a path are in the same AS. This is a policy enforcement mechanism, not a routing mechanism — BGP-X does not communicate with BGP routers or the BGP control plane.

---

## 4. Dual-Stack Traffic Separation

On a dual-stack router:

```
LAN Packet arrives
         │
         ▼
Routing Policy Engine
(domain-aware: may specify domain_segments for cross-domain path)
         │
    ┌────┴──────────────────────┐
    │                           │
    ▼                           ▼
BGP-X overlay path         Standard internet path
(enters bgpx0 TUN)         (enters WAN directly)
    │                           │
    ▼                           ▼
Onion-encrypted UDP        Plain IP packet
→ WAN interface            → WAN interface
    │                           │
    └────────┬──────────────────┘
             │
             ▼
        ISP router
             │
             ▼
    BGP-routed internet
```

Both paths use the same WAN connection. Both appear as standard internet traffic at the ISP level. BGP-X traffic is encrypted and appears as UDP/7474. ISP cannot distinguish between them.

---

## 5. Complete Non-Interaction Table

| BGP Domain | BGP-X Domain | Relationship |
|---|---|---|
| AS path selection | Overlay/cross-domain path selection | Independent |
| Prefix announcement | Node/island advertisement in unified DHT | Independent |
| BGP peering (TCP/179) | Session handshakes (UDP/7474) | Independent |
| Route filters | Exit policy | Independent |
| RPKI Route Origin Auth | Advertisement Ed25519 signatures | Independent |
| BGP communities | DHT pool membership | Independent |
| IGP (OSPF/IS-IS) | BGP-X DHT routing | Independent |
| BGP hijack detection | Node advertisement signature verification | Independent |
| Anycast routing | BGP-X ServiceID addressing | Independent |
| MPLS | N/A (operates at UDP level) | Independent |

---

## 6. BGP Events and BGP-X Effects

| BGP Event | BGP-X Effect | BGP-X Response |
|---|---|---|
| BGP route change to relay node IP | IP-level path between nodes changes; overlay sessions unaffected | None required |
| BGP hijack of relay node IP | Packets may be delivered to wrong host; ChaCha20-Poly1305 prevents decryption/modification | KEEPALIVE timeout → path rebuild |
| ISP blocks UDP 7474 | Connections to relay nodes fail | Pluggable transport; alternate ports; path rebuild using non-blocked nodes |
| ISP blocks relay node IP | Connections fail; KEEPALIVE timeout | Path rebuild with alternate nodes |
| BGP route instability | Latency spikes; possible KEEPALIVE timeout | Path quality monitoring → rebuild if degraded |
| Anycast routing (CDN) | Exit gateway connects to nearest CDN edge; transparent | None |
| RPKI route origin invalid | No direct BGP-X effect; potential BGP hijack risk | Recommend RPKI-valid IP for gateway operators |

---

## 7. Gateway Clearnet Behavior

When a BGP-X gateway forwards traffic to a clearnet destination, it creates normal internet traffic:

```
Source IP:      Gateway's public IP address
Destination IP: Clearnet destination IP
Protocol:       TCP or UDP
Port:           Destination service port (e.g., 443 for HTTPS)
Payload:        Application data (e.g., TLS-encrypted HTTP)
```

This traffic is routed by BGP like any other traffic originating from the gateway's network.

The gateway does NOT:

- Announce BGP routes for the client's IP
- Modify source/destination IPs beyond standard NAT behavior
- Inject any BGP-X-specific routing information into the public internet
- Participate in BGP peering

The gateway's clearnet behavior is functionally identical to a NAT gateway or VPN exit server — it is a standard internet host that forwards traffic on behalf of other parties.

---

## 8. BGP and Cross-Domain Routing

BGP-X's domain routing uses BGP only as transport for clearnet segments. BGP has zero involvement in:

- Mesh island internal traffic
- Direct mesh-to-mesh island bridge links
- Any routing decisions within the BGP-X overlay
- Domain bridge node operation (the bridge node's WAN uses BGP; its mesh radio does not)

**Mesh island invisibility**: An entire mesh community's internet-bound traffic appears to BGP as originating from a single IP: the bridge node's WAN address. All intra-island traffic is BGP-invisible.

**Cross-domain path in BGP**: For a path clearnet-relays → bridge → mesh, BGP sees encrypted UDP between clearnet servers (for the clearnet segment) and then nothing further (mesh segment is radio, not IP). The gap in BGP-visible traffic at the bridge node is the domain transition.

**Clearnet client accessing mesh service**: A clearnet client with no mesh hardware can reach services on a mesh island. BGP sees encrypted UDP between the client's entry node and a domain bridge node. BGP cannot see that the destination is inside a mesh island — the bridge node receives clearnet UDP and forwards via radio.

---

## 9. BGP Replacement Spectrum

BGP-X enables a spectrum of BGP-independence depending on deployment:

### Level 0: Overlay Only

BGP-X adds privacy; BGP unchanged. BGP-X overlay packets travel via BGP-routed infrastructure as ordinary UDP.

### Level 1: Intra-Mesh Island

Mesh nodes (Mode 4) communicate via radio. For intra-mesh traffic: BGP has zero involvement. 100% BGP-free for local communications.

### Level 2: Mesh + Gateway

All mesh users' internet traffic routes through gateway's ISP connection. Individual mesh users: zero direct BGP exposure. Gateway's IP is the only BGP-visible endpoint for all mesh traffic.

### Level 3: Multi-Island Bridging via Pools

Geographically separated mesh islands connected via internet relay pools. Users on any island communicate privately through BGP-X. No individual user directly exposed to BGP. Internet relay pool nodes use BGP as transport; the overlay is encrypted throughout.

### Level 4: Dense Multi-Mode Hybrid

All six deployment modes simultaneously. BGP dependence varies by user position:
- Mesh-only users: zero direct BGP exposure
- Internet-connected users: BGP as transparent transport
- Gateway users: partial (gateway mediates)

### Level 5: Wide-Area Mesh with Aerial Nodes

Elevated relay nodes on masts, aerostats, or towers connect separated mesh segments. Satellite gateways provide clearnet access without terrestrial ISP. Near-total BGP independence for intra-network communications.

### Level 6: Maximum Independence

BGP-X replaces ISP routing decisions and surveillance capability. Individual users are never directly exposed to ISP-level BGP routing for their communications.

**Honest limit**: BGP-X cannot replace physical infrastructure. Fiber, cellular towers, satellite hardware, radio spectrum — these exist at a layer below both BGP and BGP-X. BGP-X can make ISPs irrelevant from a surveillance and control perspective but cannot eliminate the physical network substrate.

**Domain routing as orthogonal dimension**:

| Domain Depth | Description | Combined with Level 2+ |
|---|---|---|
| 0 | Single domain (clearnet only) | BGP-X adds privacy; BGP unchanged |
| 1 | Clearnet + overlay | More hop diversity |
| N | N distinct routing domains | Mesh islands provide BGP-independence |

At Level 2 + Domain Depth 2: mesh island users have zero BGP exposure AND clearnet users can reach mesh services without special hardware. These properties reinforce each other.

---

## 10. Pool-Based Coverage Gap Bridging

Two mesh islands separated by distance can be connected via internet relay pools. The unified DHT cross-domain synchronization makes this automatic and transparent.

```
Island A (Lima) → [Pool: mesh-lima] → [Bridge A, WAN] →
[Pool: default (internet BGP-X relays)] →
[Bridge B, WAN] → [Pool: mesh-bogota] → Island B (Bogotá)

Individual users on either island:
  - Never directly exposed to BGP
  - Cannot see the internet bridge (transparent via pools)
  - Receive same privacy guarantees as any other BGP-X path
```

BGP sees: encrypted UDP between servers (Bridge A → clearnet relays → Bridge B). BGP cannot see that this is bridging two mesh communities. BGP cannot see source or destination within either mesh island.

The internet relay pool nodes use BGP to route their UDP traffic. From BGP's perspective: ordinary UDP datagrams between internet servers. From BGP-X perspective: encrypted relay hops in a multi-segment path.

**Bridge node DHT role**: Bridge nodes store mesh-domain node advertisements in the unified DHT on behalf of mesh-only nodes. When a bridge reconnects to the internet, it publishes updated island records. Discovery works from clearnet without any special client hardware.

---

## 11. NAT Traversal

BGP-X nodes that are behind NAT (e.g., home routers, some cloud providers) require special handling.

### 11.1 BGP-X Node Behind NAT (Relay Node)

A relay node behind NAT:

- Cannot accept inbound connections from other nodes
- Cannot advertise a reachable endpoint for the overlay
- SHOULD NOT advertise itself as a relay, entry, or exit node

BGP-X does not implement NAT traversal for relay nodes. Relay nodes MUST have a publicly reachable UDP endpoint.

### 11.2 BGP-X Client Behind NAT

BGP-X clients (not nodes) do not need a publicly reachable address. The client initiates the handshake to the entry node, and the entry node responds to the client's NAT-translated address. Standard UDP NAT traversal applies:

- The NAT translation is maintained by KEEPALIVE packets (sent every 25 seconds, randomized ±5 seconds)
- The entry node sees the client's NAT-translated IP (not the client's internal IP)

### 11.3 NAT Detection for Nodes

The node daemon attempts to determine its public IP address via:

1. Checking `public_addr` in configuration (highest priority)
2. STUN query to a well-known STUN server
3. HTTP request to an IP-reflection service

If the detected public IP differs from all local interface addresses, the node is behind NAT and will log a warning recommending the operator configure port forwarding or use a public IP host.

### 11.4 Cellular CGNAT

Cellular connections often use CGNAT. Gateway nodes on cellular should:

- Use STUN for public IP detection
- Or configure a static public IP via their cellular provider (if available)
- Or rely on relay pools rather than direct entry node operation

---

## 12. IPv4 and IPv6

BGP-X operates over both IPv4 and IPv6. The overlay protocol itself is IP-version agnostic — the transport (IPv4 or IPv6) is determined by the addresses in node advertisements.

### 12.1 Dual-Stack Operation

A BGP-X node SHOULD advertise both IPv4 and IPv6 endpoints if available. The client selects the endpoint address family based on:

- Its own network capabilities (IPv4-only, IPv6-only, or dual-stack)
- The address family of available endpoints for the selected node

### 12.2 IPv6 Path Diversity

For diversity enforcement, IPv4 and IPv6 nodes in the same ASN are treated as being in the same AS (ASN diversity check covers both address families). A path with one IPv4 node in AS12345 and one IPv6 node in AS12345 would fail the ASN diversity constraint.

This prevents trivial diversity bypass by switching address families within the same network.

### 12.3 Node Advertisement Example

```json
{
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
    { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
  ]
}
```

For domain-aware advertisements:

```json
{
  "routing_domains": [
    {
      "domain_type": "clearnet",
      "endpoints": [
        { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
        { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
      ]
    }
  ]
}
```

---

## 13. Traffic Analysis by BGP Participants

BGP participants (ISPs, IXPs, transit providers) with visibility into BGP-X traffic can observe:

- That BGP-X-using clients connect to BGP-X entry nodes (visible as UDP traffic to port 7474)
- The volume and timing of BGP-X traffic from a specific client
- That a BGP-X gateway generates UDP traffic to port 7474 (relay traffic) and TCP/UDP traffic to clearnet destinations

BGP participants CANNOT observe:

- The content of any BGP-X packet (encrypted)
- The routing within the BGP-X overlay (hidden in onion encryption)
- The relationship between a client's traffic and any specific destination
- Whether traffic is destined for a mesh island or clearnet service
- Domain transitions in cross-domain paths

This represents the BGP-X privacy boundary at the network layer. Application-layer protections (HTTPS, TLS) provide additional content privacy at the exit gateway.

---

## 14. Satellite WAN for Bridge Nodes

Bridge nodes with satellite WAN (Starlink, Iridium, etc.) participate in BGP-X exactly as fiber/cellular bridge nodes. The satellite provider uses BGP-routed internet for its backhaul — from BGP-X's perspective, this is just another clearnet transport.

**Satellite internet is clearnet**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat, OneWeb, Kuiper) provide BGP-routed IP connectivity. From BGP-X's protocol perspective, a node with Starlink WAN is a clearnet node — domain type 0x00000001. The physical medium (satellite radio vs. fiber) is invisible to the BGP-X protocol layer.

**Domain type 0x00000005** is RESERVED for a future BGP-X-native satellite network where satellites themselves run BGP-X relay software with inter-satellite links. This is NOT currently active. No commercial satellite service provides this capability.

### Configuration Example

```toml
[[routing_domains]]
domain_type = "clearnet"
endpoints = [{ "protocol": "udp", "address": "auto-detect", "port": 7474 }]
wan_provider = "starlink"
latency_class = "satellite-leo"  # 20-40ms RTT
```

Path construction uses satellite-class latency thresholds for geo plausibility scoring. Interactive traffic paths may avoid high-latency satellite segments (configurable via routing policy).

---

## 15. ECH and DNS

When an exit gateway establishes TLS connections to clearnet destinations:

- ECH DNS queries: gateway uses DoH (encrypted DNS) to fetch HTTPS records
- DoH traffic goes to clearnet (standard BGP-routed) but is encrypted
- ECH public keys are in DNS HTTPS records (public information)
- No BGP-specific concerns

ECH configuration at exit nodes:
- `ech_capable: true` in exit policy
- DoH resolver configured (default: Quad9)
- DNSSEC validation enabled

---

## 16. RPKI Recommendations

For bridge node and gateway operators: RPKI-validated IP space reduces BGP hijacking risk against the WAN endpoint.

- BGP-X overlay protection remains intact even under BGP hijack (ChaCha20-Poly1305 prevents decryption)
- But service would be disrupted until KEEPALIVE timeout → path rebuild
- RPKI reduces hijack likelihood

**Recommendation**: Bridge node and gateway operators SHOULD use IP space covered by valid RPKI Route Origin Authorizations. This is defense-in-depth, not a BGP-X protocol requirement.

BGP-X gateway operators are RECOMMENDED to:

- Use IP address space covered by valid RPKI Route Origin Authorizations (ROAs)
- Ensure their ASN is correctly registered with their RIR
- Operate on networks with BCP38 anti-spoofing filtering enabled

These measures protect the gateway's IP space from BGP hijacking, which could redirect BGP-X traffic through an adversary-controlled path at the IP level. While BGP-X's onion encryption would protect traffic content even in this scenario, routing integrity reduces the attack surface.

BGP-X does not directly interact with RPKI infrastructure.

---

## 17. Multiple WAN and Failover

Gateway nodes may have multiple WAN connections:

- **Fiber + cellular backup**: Primary fiber, failover to 4G/5G
- **Dual ISP**: Load balancing and redundancy
- **Satellite + terrestrial**: Satellite as backup for remote deployments

BGP-X overlay continues during failover. Sessions survive brief WAN transitions. KEEPALIVE timing should account for satellite-class latency when satellite is active.

---

## 18. Geographic Plausibility and BGP

Geographic plausibility scoring is an OPTIONAL reputation signal in BGP-X. It is:

- NOT required for node operation
- Applied only if a node voluntarily declares a jurisdiction
- Based on RTT measurements, not external databases
- Independent of BGP routing tables or AS path information

BGP-X nodes are NOT required to declare their jurisdiction. Nodes that do declare jurisdiction enable geo plausibility scoring; nodes that do not are unaffected.

Satellite-connected nodes are exempt from geo plausibility scoring because satellite terminal IP addresses may be geographically distant from the ground station that assigned them.

---

## 19. Honest Limits

**What BGP-X can replace**:

- ISP routing decisions
- ISP surveillance capability
- Per-user BGP exposure
- Dependency on specific ISP for privacy

**What BGP-X cannot replace**:

- Physical infrastructure (fiber, towers, cables, spectrum)
- Radio propagation physics
- Satellite orbital mechanics
- The existence of network operators who maintain physical infrastructure

BGP-X makes ISPs irrelevant as surveillance entities; it cannot make them irrelevant as physical infrastructure operators. A mesh island with no bridge node has zero internet connectivity regardless of BGP-X's capabilities.
