# BGP and BGP-X: Coexistence, Layer Relationship, and Domain Routing

**Version**: 0.1.0-draft

---

## 1. The Core Relationship

BGP-X does not participate in BGP (Border Gateway Protocol). BGP-X uses the BGP-routed internet as one of several transport layers. They operate at different layers with no direct coupling and no interaction protocol.

BGP is the routing protocol that connects the internet's Autonomous Systems. BGP-X is a privacy overlay that runs on top of BGP-routed infrastructure. BGP-X packets appear to every BGP router as ordinary encrypted UDP datagrams — opaque, indistinguishable from any other UDP traffic.

---

## 2. What BGP Is (Brief)

The Border Gateway Protocol (BGP) is the routing protocol that connects the internet's Autonomous Systems (AS). Every major network operator (ISP, cloud provider, university) runs BGP to advertise which IP address blocks they own and to exchange routing information with other operators.

BGP determines how packets travel across the internet at the infrastructure level. When your computer sends a packet to a server in another country, BGP-configured routers at each ISP along the path determine the next hop.

BGP has no privacy features. Source and destination IP addresses are visible to every router that handles a packet. BGP is optimized for reachability and stability, not privacy.

---

## 3. What BGP-X Is

BGP-X is a privacy overlay network. It routes traffic through a chain of relay nodes such that no single party can see both where traffic originates and where it is going.

BGP-X is NOT a BGP replacement. BGP-X is NOT a new routing protocol at the infrastructure level. BGP-X is a layer of software that runs on top of BGP-routed infrastructure and uses it as a transparent packet delivery service.

---

## 4. Layer Relationship

```
┌─────────────────────────────────────────────────────────────────┐
│                 BGP-X APPLICATION LAYER                         │
│          (applications, SDK, configuration client)              │
├─────────────────────────────────────────────────────────────────┤
│              BGP-X INTER-PROTOCOL DOMAIN ROUTER                 │
│         (cross-domain path construction, domain bridges)        │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X OVERLAY ROUTING LAYER                     │
│         (onion encryption, unified DHT, pools, reputation)      │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X TRANSPORT LAYER                           │
│     UDP/IP (clearnet) | WiFi mesh | LoRa | BLE | Satellite      │
│     [Satellite = clearnet transport over BGP-routed internet]   │
├─────────────────────────────────────────────────────────────────┤
│                 BGP ROUTING LAYER (clearnet only)               │
│           (ISP routing, AS-level decisions, BGP tables)         │
├─────────────────────────────────────────────────────────────────┤
│                 PHYSICAL / LINK LAYER                           │
│             (fiber, cable, radio, physical infrastructure)      │
└─────────────────────────────────────────────────────────────────┘
```

BGP occupies a layer below BGP-X. BGP-X uses BGP-routed infrastructure as its clearnet transport, the same way it uses WiFi 802.11s as its mesh transport — BGP is one transport option among several, not a peer or a constraint.

For mesh deployments, the transport layer is replaced entirely by radio (LoRa, WiFi mesh, BLE) with zero BGP involvement. A mesh island communicates internally with no BGP participation whatsoever.

**Satellite Internet in the Layer Model**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) provide internet connectivity over satellite radio. From BGP-X's perspective, these are **clearnet domain** (type 0x00000001), not a separate routing domain. They appear in the transport layer alongside fiber and cellular — same UDP/IP, different physical medium.

**Domain type 0x00000005 (bgpx-satellite)** is RESERVED for a future BGP-X-native satellite network where satellites themselves run BGP-X relay software and communicate via inter-satellite links. No current commercial satellite service provides this. Domain type 0x00000005 is reserved and not active in current deployments.

---

## 5. What BGP Sees From BGP-X Traffic

To every BGP router along the path between a BGP-X node and any other clearnet BGP-X node:

```
Source IP:      Client's WAN IP (or router's WAN IP)
Destination IP: BGP-X relay node's IP
Protocol:       UDP
Port:           7474 (default)
Payload:        Encrypted bytes (opaque)
```

BGP cannot see inside. BGP cannot determine these are BGP-X packets. BGP cannot see the actual destination of the traffic.

For cross-domain paths: when traffic traverses a clearnet segment between a domain bridge node and other clearnet relay nodes, BGP sees exactly the same — ordinary UDP between server IPs. BGP cannot determine that the traffic is destined for a mesh island service. BGP cannot see the domain bridge relationship.

From BGP's perspective, BGP-X overlay traffic is indistinguishable from any other UDP traffic on port 7474.

---

## 6. What BGP-X Sees From BGP

BGP-X treats the public internet as a best-effort packet delivery service. BGP-X:

- Knows the IP addresses of remote BGP-X nodes (from their DHT advertisements)
- Sends UDP packets to those IP addresses
- Relies on BGP-routed infrastructure to deliver those packets
- Has NO visibility into BGP routing tables, AS paths, or BGP decisions
- Does NOT participate in BGP peering
- Does NOT announce routes
- Does NOT require any ISP configuration or cooperation

BGP-X uses ASN information only from node advertisements (self-reported by node operators) for path diversity enforcement. This is NOT a BGP lookup — it's the node's self-declared ASN, indirectly verified via geographic plausibility scoring.

**Geographic Plausibility — OPTIONAL**: Geographic plausibility scoring is an OPTIONAL reputation signal. Nodes may declare a jurisdiction in their advertisement. If jurisdiction is declared, geo plausibility scoring applies, measuring RTT to verify the claimed region. If jurisdiction is NOT declared, geo plausibility scoring does NOT apply. Nodes are NOT required to declare jurisdiction. Satellite-connected nodes (latency_class = satellite-*) are exempt from geo plausibility scoring because satellite terminal IPs may be assigned from distant ground stations.

---

## 7. BGP and Cross-Domain Routing

BGP-X's cross-domain routing (clearnet ↔ mesh ↔ satellite) uses BGP as a transport for clearnet segments, with zero BGP involvement for other segment types.

### What BGP Sees in a Cross-Domain Path

For a path: clearnet relays → domain bridge → mesh island:

- BGP sees: encrypted UDP datagrams between clearnet servers (client → entry → relay → bridge node)
- BGP sees: encrypted UDP datagrams from bridge node to its upstream clearnet relay (return path)
- BGP does NOT see: anything that happens after the bridge node forwards to mesh radio
- BGP does NOT see: that the traffic is destined for a mesh island service
- BGP does NOT see: the domain bridge relationship

### Mesh Island Invisibility to BGP

A mesh island's internal traffic — all of it — is completely invisible to BGP. The entire island appears to BGP as a single server: the bridge node's WAN IP address. Every mesh island member's traffic appears to originate from and be destined to that single server IP.

This is analogous to how NAT makes an entire LAN appear as one IP — except that with BGP-X, even the content of connections to/from that IP is encrypted and opaque.

### Cross-Domain Routing as BGP-Independence Mechanism

The degree to which a deployment is BGP-independent is directly determined by which routing domains it operates in:

| Domain of user | BGP involvement for that user |
|---|---|
| Clearnet only | Full BGP transport for all hops |
| Mesh island (no bridge online) | Zero BGP involvement |
| Mesh island (bridge to clearnet overlay) | Zero direct BGP exposure; gateway mediates |
| Any-to-any cross-domain path | BGP only for the clearnet segments; zero for mesh segments |

---

## 8. Points of Complete Non-Interaction

These elements are fully independent between BGP and BGP-X:

| BGP Domain | BGP-X Domain | Relationship |
|---|---|---|
| AS path selection | Overlay path selection | Independent. BGP decides how packets travel between ISPs; BGP-X decides which relay nodes to use. |
| Prefix announcement | Node advertisement in DHT | Independent. BGP announces IP blocks; BGP-X advertises node capabilities in a private DHT. |
| BGP peering sessions | BGP-X session handshakes | Independent. BGP peers connect via TCP on port 179; BGP-X sessions use UDP and Noise-inspired handshakes. |
| Route filters | Exit policy | Independent. BGP route filters control what prefixes are accepted; BGP-X exit policies control what traffic gateways forward. |
| RPKI Route Origin Authorizations | Node advertisement signatures | Independent. RPKI validates BGP route origins; BGP-X validates node advertisements via Ed25519 signatures. |
| BGP communities | DHT pool membership | Independent. BGP communities are routing hints between ISPs; BGP-X pools are trust segments within the overlay. |
| IGP (interior routing) | BGP-X intra-overlay routing | Independent. OSPF/IS-IS route within an AS; BGP-X DHT handles relay node discovery. |
| BGP hijack detection | Node advertisement signature verification | Independent. |
| Anycast routing | BGP-X ServiceID addressing | Independent. |
| MPLS | N/A (BGP-X operates at UDP level) | Independent. |

---

## 9. Points of Indirect Interaction

These BGP-domain events affect BGP-X behavior without direct coupling:

### BGP Route Change

**BGP event**: A BGP route to a relay node's IP changes (different AS path, different latency).

**BGP-X effect**: The UDP packets between BGP-X nodes travel a different physical path. BGP-X sessions are unaffected. Latency may change slightly.

**BGP-X response**: None required. The overlay is agnostic to the underlying IP path.

### BGP Hijack of a Node's IP

**BGP event**: A malicious actor announces the IP prefix containing a BGP-X relay node, causing traffic to be routed to the wrong destination.

**BGP-X effect**: Packets destined for the relay node may be delivered to the hijacker's network instead.

**BGP-X protection**: ChaCha20-Poly1305 authentication tags prevent the hijacker from forging valid BGP-X packets. The hijacker cannot decrypt relay traffic. KEEPALIVE timeouts will detect the unresponsive node; path rebuild triggers.

**BGP-X recommendation**: Gateway operators use RPKI-valid IP space to reduce hijack risk.

### ISP Blocking BGP-X Node IP or Port 7474

**BGP event / ISP action**: ISP blocks UDP traffic to known BGP-X entry node IPs or drops port 7474 traffic.

**BGP-X effect**: Connections to affected relay nodes fail. KEEPALIVE timeouts detected.

**BGP-X response**: Path rebuilds using alternate nodes. Pluggable Transport (PT) obfuscates BGP-X traffic to avoid port-based or IP-based blocking.

**Pluggable Transport Architecture**:

PT operates at the transport layer (below BGP-X overlay, above physical):

```
BGP-X overlay packets → PT obfuscation → UDP (obfuscated) → ISP → BGP routing
```

The ISP sees obfuscated UDP traffic that does not match BGP-X signatures. BGP still routes it to the relay node. The relay node's PT layer deobfuscates before BGP-X processing.

**PT Security Model**: PT is a traffic analysis resistance mechanism, not a cryptographic security guarantee. Sufficiently sophisticated deep packet inspection (DPI) with machine learning classification may still identify patterns. Nation-state level traffic analysis over long time periods may succeed. PT raises the cost of blocking significantly.

**Built-in PT Limitations**: The built-in PT is effective against port-based blocking and simple protocol fingerprinting. It is NOT guaranteed effective against advanced DPI with machine learning or nation-state level traffic analysis.

**Recommendation**: For maximum obfuscation resistance in hostile network environments, use external PT implementations specifically designed for censorship circumvention (e.g., obfs4proxy with properly configured bridges). The built-in PT is a reasonable default for users who want basic traffic obfuscation without additional configuration.

### BGP Anycast

**BGP event**: Multiple servers announce the same IP prefix (anycast). BGP routes to the nearest.

**BGP-X interaction**: BGP-X exit nodes connecting to anycast destinations (DNS, CDNs) are unaffected — they connect to destination IP as normal. BGP's anycast routing is transparent to BGP-X. This is expected behavior.

---

## 10. Dual-Stack Traffic Separation — One WAN, Two Routing Planes

On a dual-stack router, BGP and BGP-X share the same physical WAN connection. They do not conflict because they operate at different layers:

```
LAN Packet arrives
         │
         ▼
Routing Policy Engine (Layer 3.5 — between LAN and WAN)
         │
    ┌────┴──────────────────────┐
    │                           │
    ▼                           ▼
BGP-X overlay path         Standard internet path
(enters bgpx0 TUN)         (enters WAN interface directly)
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

Both paths use the same physical WAN cable. BGP-X overlay traffic and standard traffic leave the router on the same interface. The ISP cannot distinguish between them at the IP level (BGP-X appears as ordinary UDP).

---

## 11. Why BGP-X Doesn't Need BGP Participation

BGP participants (ASes, ISPs) announce IP prefixes and exchange routing information. This is necessary for infrastructure operators who own IP space and need other operators to route traffic to them.

BGP-X relay nodes receive traffic on their existing IP addresses. They don't need to announce new prefixes because they're not providing infrastructure-level routing — they're running application-layer overlay software.

A BGP-X relay node is no different from any other internet-connected server from BGP's perspective. It has an IP address, and BGP routes traffic to it the same as to any other server.

**No ASN Required**: BGP-X nodes do not need Autonomous System Numbers. An ASN is required for BGP peering, which BGP-X does not do. A BGP-X relay node operates entirely at the application layer on top of its host's existing IP connectivity.

**No Prefix Announcement**: BGP-X nodes do not announce IP prefixes to BGP. They use their existing IP address (static, DHCP, or dynamic) and rely on their ISP's BGP announcements to make that IP reachable.

**No ISP Cooperation Required**: BGP-X works without any ISP configuration changes. A BGP-X node can be deployed on any internet-connected host with a routable IP address.

---

## 12. What Happens if an ISP Does BGP Hijacking

If an ISP or malicious actor hijacks the BGP route to a BGP-X relay node's IP:

1. **Content is protected**: ChaCha20-Poly1305 authentication prevents decryption or modification by the hijacker
2. **Session fails**: KEEPALIVE timeouts detect the node is unresponsive
3. **Path rebuilds**: Client selects alternate nodes from DHT
4. **No data leakage**: The hijacker receives BGP-X encrypted UDP datagrams they cannot decrypt

BGP hijacking cannot compromise the privacy properties of BGP-X sessions in progress. It can cause session disruption (which BGP-X handles via path rebuilding).

---

## 13. BGP Replacement Spectrum

BGP-X enables a spectrum of BGP-independence depending on deployment mode. This is orthogonal to domain routing depth — they can be combined.

### BGP-Independence Levels

**Level 0**: Pure overlay — BGP unchanged, BGP-X adds privacy only. Users gain privacy but zero BGP independence.

**Level 1**: Intra-mesh island — zero BGP involvement for local mesh traffic. The mesh community can communicate internally without any ISP. No BGP packet is generated for intra-mesh traffic.

**Level 2**: Mesh + gateway — individual mesh users have zero direct BGP exposure; all their internet traffic routes through the gateway's ISP connection. From BGP's perspective: the gateway is one server. Everything behind the gateway (the entire mesh community) is invisible.

**Level 3**: Multi-island bridging — two or more physically separated mesh islands that cannot directly connect via radio are bridged via internet relay pools. Individual users on either island have zero BGP exposure. The internet relay nodes are standard BGP-X relay nodes; BGP sees their traffic as ordinary UDP.

**Level 4**: Dense multi-mode hybrid — all modes operating simultaneously. BGP dependence varies by user position:
- Mesh-only users: zero direct BGP exposure for all traffic
- Internet-connected users: BGP as transport only (overlay encrypts all traffic)
- Gateway users: partial — WAN connection uses BGP, mesh traffic doesn't

**Level 5**: Wide-area aerial mesh — elevated relay nodes (fixed masts, tethered aerostats on high ground) connect geographically separated mesh segments without internet involvement. Satellite links (Starlink) provide clearnet gateway connectivity without requiring terrestrial ISP. BGP dependence: minimal. Satellite uses BGP-routed internet for its backhaul, but from the mesh community's perspective, the satellite link is just another WAN for the gateway node.

**Level 6**: Maximum independence — BGP-X replaces ISP routing decisions. **Honest limit**: BGP-X cannot replace the physical infrastructure that BGP-routed networks operate on (fiber optic cables, cellular towers, spectrum allocation, submarine cables). These physical elements are BGP-independent by nature but also BGP-X independent — they're physics, not software.

### Domain Routing Depth (Orthogonal Dimension)

| Domain Depth | Description |
|---|---|
| 0 | Single domain (clearnet only) |
| 1 | Clearnet + BGP-X overlay |
| 2 | Clearnet + overlay + 1 mesh island |
| N | Any combination of N distinct routing domains |

A Level 2 BGP-independence deployment at Domain Depth 2 provides: mesh users with zero BGP exposure AND clearnet users reaching mesh services without any special hardware. These two properties reinforce each other.

### Combining BGP-Independence with Domain Routing

BGP-independence level and domain routing depth can be combined:

- **Level 2 + Depth 2**: Mesh users have zero BGP exposure; clearnet users can reach mesh services without special hardware
- **Level 3 + Depth 3**: Multi-island bridging via clearnet relays; all cross-island traffic is onion-encrypted and BGP-opaque
- **Level 5 + Depth N**: Aerial/satellite mesh with cross-domain routing; near-total BGP independence for all users

**Satellite WAN in BGP-Independence Spectrum**: A mesh island using Starlink for its gateway WAN is at BGP-independence Level 2 (mesh users have zero direct BGP exposure). The satellite link uses BGP-routed internet for its backhaul, but from the mesh community's perspective, it's just a high-latency WAN connection. The physical medium (satellite radio vs. fiber) is invisible to BGP-X routing.

---

## 14. Coverage Gap Bridging via Domain Routing

Two mesh islands that cannot directly connect via radio can be bridged via the clearnet overlay, transparently and privately:

```
Mesh Island A (Lima, Peru)           Clearnet Gap           Mesh Island B (Bogotá, Colombia)
[Pool: mesh-lima]                                           [Pool: mesh-bogota]
         │                                                          │
         └──────────────────────────────────────────────────────────┘
                            bridged via
                   [clearnet BGP-X relay pool]
                   (encrypted, opaque to BGP)

User on Island A → mesh hops → Bridge A → clearnet relays → Bridge B → mesh hops → User on Island B
```

The clearnet relay pool uses BGP as transport for those hops. BGP sees only encrypted UDP between servers. BGP cannot see the end-to-end path structure or that it is bridging two mesh communities.

**Unified DHT**: The unified DHT cross-domain synchronization (performed by bridge nodes) makes Island B's nodes discoverable from Island A's DHT without any manual configuration. There is one unified DHT spanning all routing domains. Bridge nodes with clearnet endpoints serve as the physical DHT storage infrastructure accessible from the internet. Mesh-only nodes access the unified DHT via their bridge nodes' caches.

---

## 15. RPKI and BGP-X

Resource Public Key Infrastructure (RPKI) validates BGP route origin announcements. It helps prevent BGP hijacking by ensuring that only authorized ASes can announce specific IP prefixes.

BGP-X and RPKI operate at different layers and have no direct coupling:

- RPKI is a BGP-domain mechanism
- BGP-X relay operators who use RPKI-validated IP space benefit from reduced BGP hijacking risk
- BGP-X does not require RPKI and does not validate RPKI itself
- Gateway operators are **recommended** to use RPKI-valid IP space as a defense-in-depth measure

**Recommendation**: Bridge node and gateway operators SHOULD use IP space covered by valid RPKI Route Origin Authorizations. This is defense-in-depth, not a BGP-X protocol requirement.

---

## 16. IPv6 and BGP-X

BGP-X supports both IPv4 and IPv6 at the clearnet transport layer. Nodes may advertise both:

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

For diversity enforcement: IPv4 and IPv6 nodes in the same ASN are treated as the same ASN. This prevents trivial diversity bypass by switching address families within the same network.

---

## 17. Web3 and BGP-X

Web3 (blockchain protocols, DeFi, NFT platforms, cross-chain systems like OpenAccord) is an **application layer** that runs over the internet using standard protocols (HTTPS, WebSocket, libp2p). It is NOT a separate BGP-X routing domain.

From BGP-X's perspective:
- Connecting to an Ethereum RPC endpoint is clearnet (UDP/IP over BGP-routed internet)
- Connecting to a Solana validator is clearnet
- Connecting to an IPFS node via libp2p is clearnet

Web3 traffic benefits from BGP-X privacy (IP addresses hidden from RPC providers, validators, and blockchain nodes) but does not require any new BGP-X routing domain. BGP-X treats blockchain traffic identically to any other clearnet application traffic.

**Web3 on mesh islands**: A mesh island user can access Web3 services via a domain bridge node. The bridge node provides clearnet access to blockchain RPCs. The user's mesh address is hidden; only the bridge's WAN IP is visible to the RPC endpoint.

---

## 18. Honest Limits

BGP-X can make ISPs irrelevant from a surveillance and control perspective. ISP routing decisions, ISP surveillance capability, ISP-level blocking and censorship, dependency on any single ISP or jurisdiction — all can be bypassed.

BGP-X cannot replace physical infrastructure. Fiber optic cables, cellular towers, satellite hardware, radio spectrum — these exist below both BGP and BGP-X. BGP-X can make what flows through that infrastructure private; it cannot replace the infrastructure itself.

At full deployment (Level 6 BGP-independence + Domain Depth N): ISP routing decisions and surveillance capability are bypassed. Jurisdictional diversity at the relay level exceeds any single government's enforcement reach. The physical network still exists. BGP-X cannot make photons travel faster or make radio spectrum appear where there is none.

**What BGP-X CAN'T Make Private**: Physical infrastructure, radio spectrum, and the existence of traffic flows. An adversary with physical access to fiber, satellite uplinks, or radio receivers can observe that traffic exists and measure its volume. BGP-X prevents the adversary from determining who is communicating with whom and what they are saying — not from detecting that encrypted traffic is flowing.
