# BGP and BGP-X: Coexistence, Layer Relationship, and Domain Routing

**Version**: 0.1.0-draft

---

## 1. The Core Relationship

BGP-X does not participate in BGP (Border Gateway Protocol). BGP-X uses the BGP-routed internet as one of several transport layers. They operate at different layers with no direct coupling and no interaction protocol.

BGP is the routing protocol that connects the internet's Autonomous Systems. BGP-X is a privacy overlay that runs on top of BGP-routed infrastructure. BGP-X packets appear to every BGP router as ordinary encrypted UDP datagrams — opaque, indistinguishable from any other UDP traffic.

---

## 2. Layer Relationship

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

---

## 3. What BGP Sees From BGP-X Traffic

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

---

## 4. What BGP-X Sees From BGP

BGP-X treats the public internet as a best-effort packet delivery service. BGP-X:

- Knows the IP addresses of remote BGP-X nodes (from their DHT advertisements)
- Sends UDP packets to those IP addresses
- Relies on BGP-routed infrastructure to deliver those packets
- Has NO visibility into BGP routing tables, AS paths, or BGP decisions
- Does NOT participate in BGP peering
- Does NOT announce routes
- Does NOT require any ISP configuration or cooperation

BGP-X uses ASN information only from node advertisements (self-reported) for path diversity enforcement. This is NOT a BGP lookup — it's the node's self-declared ASN, indirectly verified via geographic plausibility scoring.

---

## 5. BGP and Cross-Domain Routing

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

## 6. Points of Complete Non-Interaction

| BGP Domain | BGP-X Domain | Relationship |
|---|---|---|
| AS path selection | Overlay path selection | Independent |
| Prefix announcement | Node advertisement in DHT | Independent |
| BGP peering (TCP/179) | Session handshakes (UDP/7474) | Independent |
| Route filters | Exit policy | Independent |
| RPKI Route Origin Auth | Advertisement Ed25519 signatures | Independent |
| BGP communities | DHT pool membership | Independent |
| IGP (OSPF/IS-IS) | BGP-X DHT routing | Independent |
| BGP hijack detection | Node advertisement signature verification | Independent |
| Anycast routing | BGP-X ServiceID addressing | Independent |
| MPLS | N/A (BGP-X operates at UDP level) | Independent |

---

## 7. Points of Indirect Interaction

### BGP Route Change

**BGP event**: A BGP route to a relay node's IP changes (different AS path, different latency).

**BGP-X effect**: The UDP packets between BGP-X nodes travel a different physical path. BGP-X sessions are unaffected. Latency may change slightly.

**BGP-X response**: None required. The overlay is agnostic to the underlying IP path.

### BGP Hijack of a Node's IP

**BGP event**: A malicious actor announces the IP prefix containing a BGP-X relay node.

**BGP-X effect**: Packets may be delivered to the hijacker's network.

**BGP-X protection**: ChaCha20-Poly1305 authentication tags prevent the hijacker from decrypting or modifying traffic. KEEPALIVE timeouts detect the unresponsive node. Path rebuilds using alternate nodes.

### ISP Blocking BGP-X Node IP or Port 7474

**BGP event / ISP action**: ISP blocks traffic to known BGP-X entry node IPs or drops port 7474 traffic.

**BGP-X response**: Path rebuilds using alternate nodes. Pluggable transport obfuscates BGP-X traffic to avoid port-based or IP-based blocking.

### BGP Anycast

**BGP event**: Multiple servers announce the same IP prefix (anycast). BGP routes to the nearest.

**BGP-X interaction**: BGP-X exit nodes connecting to anycast destinations (DNS, CDNs) are unaffected — they connect to destination IP as normal. BGP's anycast routing is transparent to BGP-X.

---

## 8. Dual-Stack Traffic Separation — One WAN, Two Planes

On a dual-stack router, BGP and BGP-X share the same physical WAN connection without conflict:

```
LAN Packet arrives
         │
         ▼
Routing Policy Engine
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

Both paths use the same physical WAN cable. The ISP cannot distinguish between them at the IP level — BGP-X overlay traffic appears as ordinary UDP/7474.

---

## 9. BGP Replacement Spectrum

BGP-X enables a spectrum of BGP-independence depending on deployment mode. This is orthogonal to domain routing depth — they can be combined.

### BGP-Independence Levels

**Level 0**: Pure overlay — BGP unchanged, BGP-X adds privacy only.

**Level 1**: Intra-mesh island — zero BGP involvement for local mesh traffic.

**Level 2**: Mesh + gateway — mesh users have zero direct BGP exposure.

**Level 3**: Multi-island bridging — islands connected via internet relay pools; individuals never directly exposed.

**Level 4**: Dense multi-mode hybrid — variable by position.

**Level 5**: Aerial/satellite mesh — near-total BGP independence.

**Level 6**: Maximum — BGP-X replaces ISP routing decisions; physical infrastructure remains.

### Domain Routing Depth (Orthogonal Dimension)

| Domain Depth | Description |
|---|---|
| 0 | Single domain (clearnet only) |
| 1 | Clearnet + BGP-X overlay |
| 2 | Clearnet + overlay + 1 mesh island |
| N | Any combination of N distinct routing domains |

A Level 2 BGP-independence deployment at Domain Depth 2 provides: mesh users with zero BGP exposure AND clearnet users reaching mesh services without any special hardware. These two properties reinforce each other.

---

## 10. Coverage Gap Bridging via Domain Routing

Two mesh islands that cannot directly connect via radio can be bridged via the clearnet overlay, transparently and privately.

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

The unified DHT cross-domain synchronization (performed by bridge nodes) makes Island B's nodes discoverable from Island A's DHT without any manual configuration.

---

## 11. RPKI Recommendations for Bridge/Gateway Operators

RPKI (Resource Public Key Infrastructure) validates BGP route origin announcements, reducing BGP hijacking risk. Bridge nodes and gateway nodes benefit from RPKI-valid IP space:

- Reduces risk of BGP hijacking of their WAN IP
- If hijacked: BGP-X crypto still protects content (authentication tags prevent decryption/modification)
- Service would be disrupted until KEEPALIVE timeout → path rebuild

**Recommendation**: Bridge node and gateway operators SHOULD use IP space covered by valid RPKI Route Origin Authorizations. This is defense-in-depth, not a BGP-X protocol requirement.

---

## 12. Honest Limits

BGP-X can make ISPs irrelevant from a surveillance and control perspective.

BGP-X cannot replace physical infrastructure. Fiber optic cables, cellular towers, satellite hardware, radio spectrum — these exist below both BGP and BGP-X. BGP-X can make what flows through that infrastructure private; it cannot replace the infrastructure itself.

At full deployment (Level 6 BGP-independence + Domain Depth N): ISP routing decisions and surveillance capability are bypassed. Jurisdictional diversity at the relay level exceeds any single government's enforcement reach. The physical network still exists.

---

## 13. IPv6 and BGP-X

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
