# BGP and BGP-X: Coexistence and Layer Relationship

This document precisely describes the relationship between BGP (Border Gateway Protocol) — the internet's routing protocol — and BGP-X, the privacy overlay network.

---

## What BGP Is (Brief)

The Border Gateway Protocol (BGP) is the routing protocol that connects the internet's Autonomous Systems (AS). Every major network operator (ISP, cloud provider, university) runs BGP to advertise which IP address blocks they own and to exchange routing information with other operators.

BGP determines how packets travel across the internet at the infrastructure level. When your computer sends a packet to a server in another country, BGP-configured routers at each ISP along the path determine the next hop.

BGP has no privacy features. Source and destination IP addresses are visible to every router that handles a packet. BGP is optimized for reachability and stability, not privacy.

---

## What BGP-X Is

BGP-X is a privacy overlay network. It routes traffic through a chain of relay nodes such that no single party can see both where traffic originates and where it is going.

BGP-X is NOT a BGP replacement. BGP-X is NOT a new routing protocol at the infrastructure level. BGP-X is a layer of software that runs on top of BGP-routed infrastructure and uses it as a transparent packet delivery system.

---

## Layer Relationship

```
┌─────────────────────────────────────────────────────────────────┐
│                 BGP-X APPLICATION LAYER                         │
│          (your applications, SDK, configuration client)         │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X OVERLAY ROUTING LAYER                     │
│         (path selection, onion encryption, DHT, pools)          │
├─────────────────────────────────────────────────────────────────┤
│                 BGP-X TRANSPORT LAYER                           │
│            (UDP/IP datagrams over BGP-routed internet)          │
│    [For mesh deployments: WiFi mesh, LoRa, BLE, Ethernet P2P]  │
├─────────────────────────────────────────────────────────────────┤
│                 BGP ROUTING LAYER                               │
│           (ISP routing, AS-level decisions, BGP tables)         │
├─────────────────────────────────────────────────────────────────┤
│                 PHYSICAL / LINK LAYER                           │
│             (fiber, cable, radio, physical infrastructure)      │
└─────────────────────────────────────────────────────────────────┘
```

BGP-X occupies the layers above BGP. BGP operates below BGP-X. They have no direct interaction.

---

## What BGP Sees From BGP-X Traffic

From every BGP router along the path between a BGP-X client and a BGP-X relay node, all BGP-X packets appear as:

```
Source IP:      Client's WAN IP (or router's WAN IP)
Destination IP: BGP-X relay node's IP
Protocol:       UDP
Port:           7474 (default)
Payload:        Encrypted bytes (opaque)
```

BGP cannot see inside the encrypted payload. BGP cannot determine that these are BGP-X overlay packets. BGP cannot see the actual destination of the traffic (which is hidden inside the encrypted overlay).

To BGP-level network operators (ISPs, backbone providers), BGP-X overlay traffic is indistinguishable from any other UDP traffic on port 7474.

---

## What BGP-X Sees From BGP

BGP-X treats the public internet as a best-effort packet delivery service. BGP-X:

- Knows the IP addresses of remote BGP-X nodes (from their DHT advertisements)
- Sends UDP packets to those IP addresses
- Relies on BGP-routed infrastructure to deliver those packets
- Has NO visibility into BGP routing tables, AS paths, or BGP decisions
- Does NOT participate in BGP peering
- Does NOT announce routes
- Does NOT require any ISP configuration or cooperation

BGP-X uses ASN information only from node advertisements (self-reported by node operators) for path diversity enforcement. This is NOT a BGP lookup — it's the node's self-declared ASN, verified indirectly via geographic plausibility scoring.

---

## Points of Complete Non-Interaction

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

---

## Points of Indirect Interaction

These BGP-domain events affect BGP-X behavior without direct coupling:

### BGP Route Change

**BGP event**: A BGP route to a relay node's IP changes (different AS path, different latency).

**BGP-X effect**: The UDP packets between two BGP-X relay nodes now travel a different physical path. BGP-X sessions are unaffected — the sessions remain established. Packet delivery may have slightly different latency.

**BGP-X response**: None required. The overlay is agnostic to the underlying IP path.

### BGP Hijack of a Node's IP

**BGP event**: A malicious actor announces the IP prefix containing a BGP-X relay node, causing traffic to be routed to the wrong destination.

**BGP-X effect**: Packets destined for the relay node may be delivered to the hijacker's network instead.

**BGP-X protection**: ChaCha20-Poly1305 authentication tags prevent the hijacker from forging valid BGP-X packets. The hijacker cannot decrypt relay traffic. KEEPALIVE timeouts will detect the unresponsive node; path rebuild triggers.

**BGP-X recommendation**: Gateway operators use RPKI-valid IP space to reduce hijack risk.

### ISP Blocking BGP-X Node IP or Port 7474

**BGP event / ISP action**: ISP blocks UDP traffic to known BGP-X node IPs or drops port 7474 traffic.

**BGP-X effect**: Connections to affected relay nodes fail. KEEPALIVE timeouts detected.

**BGP-X response**: Path rebuilt using alternate nodes. Pluggable transport can obfuscate BGP-X traffic to avoid port-based or IP-based blocking.

### BGP Anycast

**BGP event**: Multiple servers announce the same IP prefix (anycast). BGP routes to the nearest.

**BGP-X interaction**: Some clearnet services (DNS servers, CDNs) use anycast. When a BGP-X exit node connects to an anycast destination, BGP determines which physical server receives the connection. BGP-X is unaware of anycast — it sees only the destination IP. This is expected behavior.

---

## Dual-Stack Traffic Separation — One WAN, Two Routing Planes

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

## Why BGP-X Doesn't Need BGP Participation

BGP participants (ASes, ISPs) announce IP prefixes and exchange routing information. This is necessary for infrastructure operators who own IP space and need other operators to route traffic to them.

BGP-X relay nodes receive traffic on their existing IP addresses. They don't need to announce new prefixes because they're not providing infrastructure-level routing — they're running application-layer overlay software.

A BGP-X relay node is no different from any other internet-connected server from BGP's perspective. It has an IP address, and BGP routes traffic to it the same as to any other server.

---

## What Happens if an ISP Does BGP Hijacking

If an ISP or malicious actor hijacks the BGP route to a BGP-X relay node's IP:

1. **Content is protected**: ChaCha20-Poly1305 authentication prevents decryption or modification by the hijacker
2. **Session fails**: KEEPALIVE timeouts detect the node is unresponsive
3. **Path rebuilds**: Client selects alternate nodes from DHT
4. **No data leakage**: The hijacker receives BGP-X encrypted UDP datagrams they cannot decrypt

BGP hijacking cannot compromise the privacy properties of BGP-X sessions in progress. It can cause session disruption (which BGP-X handles via path rebuilding).

---

## BGP Replacement Spectrum

BGP-X enables a spectrum of BGP-independence depending on deployment mode:

### Level 0: Overlay Only

BGP-X runs as a pure overlay. BGP is the full transport. ISP sees encrypted UDP. BGP is completely unchanged. Users gain privacy but zero BGP independence.

### Level 1: Intra-Mesh Island

BGP-X mesh nodes (Mode 4) communicate via WiFi 802.11s or LoRa radio. For intra-mesh traffic, BGP has no involvement whatsoever. 100% BGP-free for local communications.

The mesh community can communicate internally without any ISP. No BGP packet is generated for intra-mesh traffic.

### Level 2: Mesh + Gateway

Mesh community gains clearnet access via a gateway node (Mode 5). Individual mesh users have zero direct BGP exposure — all their internet traffic routes through the gateway's ISP connection. The gateway is the only BGP-visible endpoint for the entire community.

From BGP's perspective: the gateway is one server. Everything behind the gateway (the entire mesh community) is invisible.

### Level 3: Multi-Island Bridging via Pools

Two or more physically separated mesh islands that cannot directly connect via radio. A BGP-X pool spanning internet relay nodes bridges the gap:

```
[Island A Mesh] → [Pool: mesh-a] → [Pool: default (internet)] → [Pool: mesh-b] → [Island B Mesh]
```

The internet relay pool is used only to bridge the coverage gap. Traffic is onion-encrypted throughout. Individual users on either island have zero BGP exposure. The internet relay nodes are standard BGP-X relay nodes; BGP sees their traffic as ordinary UDP.

The pool-based bridge is transparent to users. The DHT gateway synchronization makes Island B's nodes discoverable from Island A's DHT without users knowing about the architecture.

### Level 4: Dense Multi-Mode Hybrid

All modes operating simultaneously. BGP dependence varies by user position:

- Mesh-only users: zero direct BGP exposure for all traffic
- Internet-connected users: BGP as transport only (overlay encrypts all traffic)
- Gateway users: partial — WAN connection uses BGP, mesh traffic doesn't

### Level 5: Wide-Area Aerial Mesh

Elevated relay nodes (fixed masts, tethered aerostats on high ground) connect geographically separated mesh segments without internet involvement. Satellite links (Starlink) provide clearnet gateway connectivity without requiring terrestrial ISP.

BGP dependence: minimal. Satellite uses BGP-routed internet for its backhaul, but from the mesh community's perspective, the satellite link is just another WAN for the gateway node.

### Level 6: Maximum Independence (Honest Limit)

**What BGP-X replaces**: ISP routing decisions, ISP surveillance capability, ISP-level blocking and censorship, dependency on any single ISP or jurisdiction.

**What BGP-X cannot replace**: The physical infrastructure that BGP-routed networks operate on (fiber optic cables, cellular towers, spectrum allocation, submarine cables). These physical elements are BGP-independent by nature but also BGP-X independent — they're physics, not software.

BGP-X can make ISPs irrelevant from a surveillance and control perspective, but cannot eliminate the need for physical connectivity at some level.

---

## Pool-Based Coverage Gap Bridging

The most important architectural capability enabled by pools and the gateway DHT cross-domain sync:

Two mesh islands that cannot directly connect via radio can be bridged via internet relay pools, transparently and privately.

```
Mesh Island A (Peru)          Coverage Gap          Mesh Island B (Colombia)
[Pool: mesh-peru]                                   [Pool: mesh-colombia]
         │                                                   │
         └──────────────────────────────────────────────────┘
                   bridged by
         [Pool: default (internet BGP-X relays)]

User on Island A → mesh hops → Gateway A → internet relay → Gateway B → mesh hops → User on Island B
```

This path is constructed automatically by path selection when the client's path config specifies the three pools. The user sees no difference from communicating with a locally-reachable mesh node.

The BGP-X DHT cross-domain synchronization (performed by gateway nodes) makes Island B's nodes appear in Island A's DHT, making this path constructible without any manual configuration by end users.

---

## RPKI and BGP-X

Resource Public Key Infrastructure (RPKI) validates BGP route origin announcements. It helps prevent BGP hijacking by ensuring that only authorized ASes can announce specific IP prefixes.

BGP-X and RPKI operate at different layers and have no direct coupling:

- RPKI is a BGP-domain mechanism
- BGP-X relay operators who use RPKI-validated IP space benefit from reduced BGP hijacking risk
- BGP-X does not require RPKI and does not validate RPKI itself
- Gateway operators are **recommended** to use RPKI-valid IP space as a defense-in-depth measure

---

## ISP Traffic Blocking and Pluggable Transport

ISPs can observe that a user is sending UDP traffic to port 7474, which may be associated with BGP-X. ISPs can block this.

**Pluggable Transport (PT)** addresses this by obfuscating BGP-X traffic to look like other protocols (obfs4-style: random-length padding + stream cipher). The obfuscated traffic does not contain recognizable BGP-X patterns.

PT operates at the transport layer (below BGP-X overlay, above physical):

```
BGP-X overlay packets → PT obfuscation → UDP (obfuscated) → ISP → BGP routing
```

The ISP sees obfuscated UDP traffic that does not match BGP-X signatures. BGP still routes it to the relay node. The relay node's PT layer deobfuscates before BGP-X processing.

PT is a traffic analysis resistance mechanism, not a cryptographic security guarantee. Sufficiently sophisticated deep packet inspection may still identify patterns. PT raises the cost of blocking significantly.
