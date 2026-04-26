# BGP-X Cross-Domain Routing Guide

This document explains BGP-X's inter-protocol routing model — how to route between the public internet, the BGP-X overlay, and mesh networks.

---

## 1. The Three Equal Domains

BGP-X treats three network types as equal first-class citizens:

### Clearnet
The BGP-routed public internet. Every device with an ISP connection is in the clearnet domain. Standard IP addressing, standard TCP/UDP.

### BGP-X Overlay
The BGP-X onion-encrypted routing layer. Spans all physical transports — UDP over internet, WiFi mesh, LoRa radio. The overlay is what provides privacy and multi-hop routing.

### Mesh Islands
Named communities of nodes communicating via local radio (WiFi 802.11s, LoRa, Bluetooth). A mesh island is an independent, addressable network that can operate without ISP connectivity.

**None of these is secondary.** A client in any domain can reach a service in any other domain. The client's entry point — whether they have an ISP connection, a BGP-X router, or just a LoRa radio — does not restrict what they can reach.

---

## 2. What This Means Practically

### You Are on Standard Internet (No BGP-X Router)
You run the BGP-X daemon on your laptop. You have a normal ISP connection. You can:
- Route through the BGP-X overlay for privacy (same as before)
- Reach services hosted inside a mesh island (new)
- Use mesh relay nodes as path hops (new)
- Configure paths that traverse multiple domains in any order (new)

### You Are on a Mesh Island (No ISP)
Your community has BGP-X nodes communicating via WiFi mesh and LoRa, plus one gateway node with satellite internet. You can:
- Communicate privately with other mesh island members (always worked)
- Route through the BGP-X overlay to clearnet services (via the gateway)
- Reach other mesh islands (via clearnet or direct LoRa bridges)
- Use a path that goes mesh → clearnet → another mesh → clearnet

### You Have Both (BGP-X Router + Mesh)
Full flexibility. Your router policy can route different traffic through different domain sequences based on sensitivity, latency, and jurisdiction requirements.

---

## 3. Domain Bridge Nodes

A **Domain Bridge Node** is any BGP-X node with endpoints in two or more routing domains.

```
                    Domain Bridge Node
                          │
         ┌────────────────┴────────────────┐
         │                                 │
   clearnet endpoint               mesh radio
   203.0.113.1:7474                WiFi mesh / LoRa
   (Internet-accessible)           (Radio-accessible)
         │                                 │
   Internet world                  Mesh island world
```

Bridge nodes are how traffic crosses from one domain to another. They are discovered via the unified BGP-X DHT — the same DHT that nodes in all domains use.

### Bridge Node Types

| Bridge | What It Does |
|---|---|
| Mesh Gateway | Connects a mesh island to the clearnet internet |
| Mesh-to-Mesh Bridge | Connects two mesh islands (can be direct radio or via internet) |
| Satellite Gateway | Connects satellite connectivity to clearnet or mesh |
| Clearnet-to-Clearnet | Standard relay/exit (existing concept) |

---

## 4. Cross-Domain Path Examples

### Example A: Clearnet Client → Mesh Service

A journalist on standard internet wants to reach a resistance group's internal service hosted on a mesh island.

```
Path:
  [Journalist's Device, clearnet]
      → clearnet relay 1 (EU)
      → clearnet relay 2 (AP)
      → clearnet relay 3 (SA)
      → DOMAIN BRIDGE NODE (clearnet + mesh:resistance-island-1)
      → mesh relay 1 (within island)
      → mesh relay 2 (within island)
      → [Resistance service, mesh:resistance-island-1]
```

The journalist has no BGP-X router. No mesh radio. Standard laptop. The domain bridge node handles the translation to mesh transport. The mesh service never sees the journalist's IP address.

### Example B: Mesh Client → Clearnet Service

A community mesh user wants to browse the web:

```
Path:
  [Community mesh user, mesh:rural-kenya-1]
      → mesh relay 1 (LoRa)
      → mesh relay 2 (WiFi mesh)
      → DOMAIN BRIDGE NODE (mesh:rural-kenya-1 + clearnet, satellite WAN)
      → clearnet relay 1 (NA)
      → clearnet exit node (EU)
      → [Destination website]
```

The community member has no ISP. Their internet access flows through the mesh → bridge node → satellite → clearnet.

### Example C: Mesh-to-Mesh via Overlay

Two geographically separated mesh islands communicate:

```
Path:
  [User on mesh:lima-district-1]
      → mesh relays within Lima island (3 hops)
      → DOMAIN BRIDGE NODE (mesh:lima-1 + clearnet)
      → clearnet relays (3 hops, for overlay mixing)
      → DOMAIN BRIDGE NODE (clearnet + mesh:bogota-district-3)
      → mesh relays within Bogotá island (2 hops)
      → [Service on mesh:bogota-district-3]
```

Neither island member is exposed to the public internet directly.

### Example D: Ultra-High-Security Cross-Domain

Maximum hops, maximum jurisdictional diversity:

```
Path:
  [Client, clearnet]
      → clearnet relay (EU-1)
      → clearnet relay (AP-1)
      → DOMAIN BRIDGE NODE (clearnet → mesh:island-A)
      → mesh relay A-1 (WiFi)
      → mesh relay A-2 (LoRa)
      → DOMAIN BRIDGE NODE (mesh:island-A → clearnet)
      → clearnet relay (SA-1)
      → clearnet exit node (AF-1)
      → [Destination]
```

This path traverses clearnet → mesh → clearnet. Three domain segments, two domain transitions. Unlimited hop count — the protocol imposes no cap.

### Example E: Multi-Island Mesh Network

Three mesh islands form a community network. All communicate through the BGP-X overlay:

```
Lima island ──[bridge]──→ Clearnet overlay ──[bridge]──→ Bogotá island
     │                                                          │
     └──────────────────[bridge]──────────────────────→ Cusco island
     (if Lima-Cusco have direct LoRa bridge: direct mesh-to-mesh)
```

---

## 5. Configuring Cross-Domain Paths

### Via SDK

```rust
use bgpx_sdk::{Client, PathConfig, SegmentConfig, RoutingDomain, DomainTransition};

let client = Client::connect_auto().await?;

// Clearnet client reaching mesh island service
let stream = client.connect_stream_with_path(
    "bgpx-mesh://lima-district-1/service-id-hex",
    PathConfig {
        segments: vec![
            SegmentConfig {
                domain: RoutingDomain::Clearnet,
                pool: "bgpx-default".into(),
                hops: 3,
                is_exit: false,
                ..Default::default()
            },
            SegmentConfig {
                domain: RoutingDomain::Mesh { island_id: "lima-district-1".into() },
                hops: 2,
                is_exit: false,
                domain_transition: Some(DomainTransition::Auto),
                ..Default::default()
            },
        ],
        ..Default::default()
    },
    Default::default(),
).await?;
```

```rust
// High-security: clearnet → mesh → clearnet
let stream = client.connect_stream_with_path(
    "destination.example.com:443",
    PathConfig {
        segments: vec![
            SegmentConfig {
                domain: RoutingDomain::Clearnet,
                pool: "bgpx-default".into(),
                hops: 2,
                ..Default::default()
            },
            SegmentConfig {
                domain: RoutingDomain::Mesh { island_id: "trusted-relay-island".into() },
                hops: 2,
                domain_transition: Some(DomainTransition::Auto),
                ..Default::default()
            },
            SegmentConfig {
                domain: RoutingDomain::Clearnet,
                pool: "private-exits".into(),
                hops: 1,
                is_exit: true,
                domain_transition: Some(DomainTransition::Auto),
                ..Default::default()
            },
        ],
        ..Default::default()
    },
    Default::default(),
).await?;
```

### Via Routing Policy

```toml
# In routing_policy.toml — route sensitive destinations through mesh segment

[[rules.destination]]
destination_domain = ["*.sensitive.com"]
action = "bgpx"
path = {
    domain_segments = [
        { domain = "clearnet", pool = "bgpx-default", hops = 2 },
        { domain = "mesh:trusted-island-1", hops = 2, domain_transition = "auto" },
        { domain = "clearnet", pool = "private-exits", hops = 1, exit = true, domain_transition = "auto" }
    ],
    require_ech = true
}
```

### Via CLI

```bash
# Build cross-domain path
bgpx-cli paths build \
    --domain-segments "clearnet:2,mesh:lima-district-1:3,clearnet:1:exit"

# List available bridge nodes for a domain pair
bgpx-cli domains bridges --from clearnet --to "mesh:lima-district-1"

# Check mesh island status
bgpx-cli domains island-status --island-id lima-district-1

# Test cross-domain connectivity
bgpx-cli domains test --from clearnet --to "mesh:lima-district-1"
```

---

## 6. N-Hop Unlimited

BGP-X imposes no maximum hop count at any level.

| Level | Maximum |
|---|---|
| Total path hops | None (protocol) |
| Hops within one domain segment | None (protocol) |
| Domain transitions per path | None (protocol) |
| Mesh relay hops within an island | None (protocol) |
| Pool segments per path | None (protocol) |

Practical defaults:
- 4 hops: single-domain clearnet
- 6–8 hops: cross-domain 2-segment
- 8–12 hops: cross-domain 3+ segments
- Any count: application-specified

The only minimum is 3 hops (absolute protocol minimum across any domain).

**Why this matters**: high-security applications may want 15-20 hops across multiple domains and mesh islands. BGP-X supports this without special configuration — just specify the segments and hops you need.

---

## 7. Offline Mesh Islands

A mesh island without any active bridge node to other domains is in **offline mode**:

- All intra-island traffic works normally
- No connectivity to other domains
- Island advertisement in local mesh DHT only (not in unified internet DHT)

When a bridge node comes online (a vehicle drives through with a satellite connection, a temporary internet link is established):

1. Bridge node publishes island advertisement to unified DHT
2. Island nodes become discoverable from clearnet within DHT TTL
3. Cross-domain paths to/from the island become constructible
4. When the bridge goes offline: island returns to offline mode

This supports **opportunistic connectivity**: mesh islands can receive/send traffic during windows of bridge availability, operating independently the rest of the time. Messages queued during offline mode (if store-and-forward is enabled) deliver when the bridge reconnects.

---

## 8. Privacy Properties

### What Cross-Domain Means for Anonymity

Cross-domain paths provide the same split-knowledge guarantees as single-domain paths:

- No relay node in any domain can see both the source IP and the final destination
- Domain bridge nodes see only their immediate predecessor in one domain and successor in the other
- path_id enables return routing across domain boundaries without revealing path composition
- A global passive adversary would need to observe traffic in multiple domains simultaneously

### Additional Properties Unique to Cross-Domain

**Mesh islands as traffic mixers**: a path that enters a mesh island and re-emerges into clearnet creates a domain boundary that even a clearnet-observing adversary cannot cross without also observing mesh radio traffic. This provides an additional layer of isolation beyond standard multi-hop routing.

**No mandatory BGP-X router**: clearnet users access mesh-hosted services without any special hardware. The privacy is provided by the domain bridge node, not by the client's network equipment.

**Offline island confidentiality**: a mesh island without a bridge cannot be accessed from outside at all. The strongest isolation is complete disconnection.

---

## 9. Operating a Domain Bridge Node

If you operate a node with both internet and mesh connectivity, you are automatically a domain bridge node when you configure both routing domains.

Minimum requirements:
- BGP-X daemon with both domains configured
- Reliable connectivity in both domains (high uptime required)
- Declared in node advertisement (automatic from config)
- Published in unified DHT bridge pair index (automatic from daemon)

Multiple domain bridge nodes per island pair: strongly recommended. If only one bridge node exists and it fails, the island loses cross-domain connectivity. Two or more independent bridge nodes (different operators, different locations) provide resilience.

```toml
# Domain bridge node configuration
[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474

[[routing_domains]]
domain_type = "mesh"
island_id = "lima-district-1"
enabled = true
transports = ["wifi_mesh", "lora"]
wifi_mesh_interface = "mesh0"
lora_interface = "/dev/ttyUSB0"

[domain_bridge]
enabled = true
bridges = [
    { from = "clearnet", to = "mesh:lima-district-1" }
]
```

---

## 10. Summary: What Changed From Single-Domain BGP-X

| Property | Before | After |
|---|---|---|
| Entry points | Clearnet or mesh (separate) | Three equal entry points |
| Cross-domain routing | Limited to mesh↔internet gateway | Any domain to any domain in any order |
| Hop count limit | Protocol-soft caps | None at any level |
| Mesh access (clearnet client) | Required BGP-X router with mesh radio | Standard BGP-X daemon sufficient |
| DHT | Two separate DHTs with gateway sync | One unified logical DHT |
| Path diversity | Single-domain diversity constraints | Cross-domain diversity (same ASN/operator constraints apply across all domains) |
| Return routing | path_id within one domain | path_id maintained across domain boundaries |
