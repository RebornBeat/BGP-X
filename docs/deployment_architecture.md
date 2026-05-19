# BGP-X Deployment Architecture

**Version**: 0.1.0-draft

This document describes the complete deployment architecture of BGP-X — all six deployment modes, how BGP and BGP-X coexist, the three local interfaces, the application type model, and guidance for choosing the right deployment for your situation.

---

## Canonical Architecture Statement

**BGP-X is router-level network infrastructure, not a per-application tool.**

The BGP-X daemon (`bgpx-node`) runs on a network router and is the single BGP-X routing stack for an entire network. Every other BGP-X component — SDK applications, the configuration client, management GUIs — is a client of this daemon. They do not implement their own routing, handshakes, or cryptography.

BGP-X is also an **inter-protocol domain router**. It routes traffic across routing domains — clearnet, BGP-X overlay, and mesh islands — in any combination, any order, with no protocol-enforced maximum on total hops or domain transitions.

This architecture means:

- Standard applications require zero modification to be protected
- All devices on the LAN are automatically protected when they connect to the router
- Application developers use the SDK to connect to the daemon — not to build their own overlay stack
- The configuration client (bgpx-cli) is a management tool, not a routing stack

---

## Routing Domains — Foundational Concept

BGP-X routes traffic across **routing domains**. A routing domain is a named network environment with its own transport characteristics.

**Defined domains**:

| Domain | Type ID | Description |
|---|---|---|
| Clearnet | 0x00000001 | BGP-routed internet — including satellite internet (Starlink, Iridium, Inmarsat, HughesNet, Viasat) |
| BGP-X Overlay | 0x00000002 | Onion-encrypted relay network (virtual domain spanning all physical transports) |
| Mesh Island | 0x00000003 | Named mesh island (WiFi 802.11s, LoRa, BLE, Ethernet P2P) |
| LoRa Regional | 0x00000004 | LoRa-only regional channel plan |
| bgpx-satellite | 0x00000005 | **RESERVED** for future BGP-X-native satellite network; NOT active in current deployments |

**Any-to-any routing**: A path can traverse any combination of domains in any order. A clearnet client can reach a mesh island service. A mesh island user can reach clearnet via a gateway. Two islands can connect via the overlay.

**Domain bridge nodes**: Any BGP-X node with endpoints in 2+ routing domains is a domain bridge. It bridges traffic between those domains without re-encrypting inner onion layers.

**N-hop unlimited**: There is no protocol maximum on total hops or domain transitions. Practical defaults exist (4 hops for single-domain clearnet, 6-8 for cross-domain), but these are configurable, not enforced.

---

## Six Deployment Modes

---

### Mode 1: Dual-Stack Router (BGP + BGP-X)

**What it is**: Standard router with ISP WAN connection. BGP-X daemon runs alongside the standard network stack. The routing policy engine decides per-flow which path to use.

**Hardware**: BGP-X Router v1.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         DUAL-STACK ROUTER                              │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    STANDARD NETWORK STACK                         │  │
│  │  DHCP, NAT, standard firewall, standard routing table            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       BGP-X DAEMON                               │  │
│  │  Path Manager, Session Manager, Onion Engine, DHT, Pool Manager  │  │
│  │  PT Engine, Reputation System, Geographic Plausibility           │  │
│  │                                                                   │  │
│  │   TUN (bgpx0) ── SDK Socket ── Control Socket                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                   ROUTING POLICY ENGINE                           │  │
│  │  Evaluates each packet/flow:                                      │  │
│  │    → BGP-X overlay (bgpx0)         per device, per destination,   │  │
│  │    → Standard BGP (WAN)            per protocol, per pool config  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  WAN: eth0 → ISP → BGP-routed internet                                │
│  LAN: eth1-4, wlan0 → devices on local network                        │
└────────────────────────────────────────────────────────────────────────┘
```

**BGP role**: Full — BGP routes all WAN traffic including BGP-X overlay packets (which appear as ordinary UDP/7474).

**BGP-X role**: Privacy overlay for selected traffic; routing policy decides which traffic gets overlay protection.

**When to use**:
- Households where some devices need low latency (gaming consoles, smart TVs)
- Mixed-use environments where selective bypass is desired
- Situations where certain destinations must appear to come from your ISP IP

---

### Mode 2: BGP-X Only Router

**What it is**: Same hardware, same daemon, different routing policy configuration. All LAN traffic is forced through BGP-X. No bypass option for LAN devices.

**Hardware**: BGP-X Router v1.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        BGP-X ONLY ROUTER                               │
│                                                                        │
│  Standard network services (DHCP, DNS, NAT) still present for LAN    │
│  All LAN-originated traffic → bgpx0 → BGP-X overlay                  │
│                                                                        │
│  WAN still uses BGP-routed internet as transport for BGP-X overlay   │
│  (BGP is the transport layer; LAN devices cannot directly access WAN) │
└────────────────────────────────────────────────────────────────────────┘
```

**Key distinction**: LAN devices have no path to the internet that doesn't go through BGP-X. The WAN connection still uses BGP-routed infrastructure — this is unavoidable since BGP-X runs on top of the internet — but from the LAN perspective, all traffic goes through the overlay.

**When to use**:
- Maximum privacy enforcement
- Environments where accidental bypass must be impossible
- Shared networks (organization, community) where all users should be protected

---

### Mode 3: Standalone Device

**What it is**: BGP-X daemon running on the user's device itself. Protects only that device's traffic.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         USER DEVICE                                    │
│                                                                        │
│  BGP-X daemon runs locally (same daemon code, different config scope) │
│  TUN interface (bgpx0) on device itself                               │
│  SDK socket on device (local apps)                                    │
│  Control socket on device (local CLI)                                 │
│                                                                        │
│  Protects: only this device                                           │
│  Does not protect: other devices on same network                      │
└────────────────────────────────────────────────────────────────────────┘
```

The daemon code is identical for router and standalone deployment. The difference is configuration scope (one device vs whole network) and which interfaces are managed.

**Geographic Plausibility**: The standalone node may optionally declare a jurisdiction. If declared, geo plausibility scoring applies. If NOT declared, geo plausibility scoring does NOT apply. Jurisdiction declaration is OPTIONAL.

**When to use**:
- No router access
- Travel
- Development and testing
- High-security single device

---

### Mode 4: BGP-X Mesh Node

**What it is**: A BGP-X router with NO ISP WAN connection. Communicates with other BGP-X nodes via local mesh transport (WiFi 802.11s, LoRa radio, Bluetooth BLE, or wired Ethernet point-to-point).

**Hardware**: BGP-X Node v1 or BGP-X Router v1.

**Key clarification**: BGP-X Node v1 is a FULL BGP-X routing node. It runs the complete bgpx-node daemon — the same software as the Router v1. The difference is form factor and deployment context, not capability. It is NOT a "lite" or partial node.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         MESH NODE                                      │
│                                                                        │
│  BGP-X daemon (same daemon code as Router v1):                        │
│    - No WAN configuration                                             │
│    - Mesh transport interfaces (WiFi mesh, LoRa radio, BLE)           │
│    - Beacon broadcast for DHT bootstrap                               │
│    - Full DHT participation via unified DHT                           │
│    - Relay role (entry + discovery + relay)                          │
│    - No exit role (no clearnet access without gateway)               │
│                                                                        │
│  LAN: provides network access to connected devices (if configured)    │
│  Mesh: WiFi 802.11s and/or LoRa radio connected to other mesh nodes  │
│  No WAN: zero ISP dependency                                          │
└────────────────────────────────────────────────────────────────────────┘
```

**BGP role**: None. This node has zero connection to BGP-routed infrastructure.

**Clearnet access**: Only available if a gateway node (Mode 5) is reachable through the mesh.

**DHT operation**: The mesh node participates in the **unified DHT**. There is no separate "mesh DHT." Mesh-only nodes access the unified DHT via their bridge/gateway nodes, which store and serve DHT records on their behalf.

**When to use**:
- Remote communities without ISP access
- Disaster recovery when ISP is unavailable
- Privacy-sensitive local communications
- Community mesh networks
- Range extension for existing mesh island

---

### Mode 5: BGP-X Gateway Node — Domain Bridge

**What it is**: A BGP-X router with BOTH mesh transport interfaces AND an ISP WAN connection. Bridges the mesh network to the clearnet internet. Also called a "domain bridge node."

**Hardware**: BGP-X Router v1 or BGP-X Node v1 (with WAN option). BGP-X Gateway v1 is provider-grade infrastructure for high-throughput operator deployments.

**Key concept**: A gateway node is a DOMAIN BRIDGE — it bridges the mesh routing domain to the clearnet routing domain. This is the mechanism by which mesh island users reach the internet, and clearnet users reach mesh island services.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         GATEWAY NODE                                   │
│                        (Domain Bridge)                                 │
│                                                                        │
│  BGP-X daemon with:                                                   │
│    - Mesh transport interfaces (receive from mesh community)          │
│    - WAN interface (ISP connection for clearnet)                      │
│    - Domain bridge configuration (clearnet ↔ mesh)                   │
│    - Exit policy enforcement (if providing clearnet exit)             │
│    - DoH DNS resolver with DNSSEC + ECS stripping                    │
│    - ECH support when destination publishes ECH config                │
│    - Publishes DOMAIN_ADVERTISE for bridge capability                 │
│    - Publishes MESH_ISLAND_ADVERTISE for served islands              │
│                                                                        │
│  Participates in unified DHT                                          │
│  Clearnet users can discover mesh island services via this bridge    │
│  Mesh users can reach clearnet via this bridge's exit                │
└────────────────────────────────────────────────────────────────────────┘
```

**Architecture**:
```
Mesh Community → [mesh hops] → Gateway → [clearnet relays] → Exit → Destination
                                  │
                           ISP WAN connection
                           BGP-routed internet
```

**From the destination's perspective**: traffic comes from the gateway's exit node IP (not any individual mesh user's IP).

**From BGP's perspective**: the gateway is a standard internet host. BGP cannot see inside BGP-X overlay packets.

**Clearnet users accessing mesh services**: A clearnet client with NO mesh radio can reach services inside a mesh island by constructing a cross-domain path that includes a DOMAIN_BRIDGE hop through this gateway. The gateway receives clearnet UDP packets and forwards them into the mesh via its radio.

**DHT**: Single unified DHT. The gateway stores mesh-domain node advertisements in the unified DHT on behalf of mesh-only nodes that cannot directly reach internet DHT nodes.

**Satellite WAN option**: A gateway in a remote location without fiber/cellular can use satellite internet (Starlink, Iridium) as its WAN. From BGP-X's perspective, this is clearnet with satellite-class latency. See `satellite_architecture.md`.

**When to use**:
- Community mesh needs internet access for some traffic
- Multiple gateways in different jurisdictions for redundancy and diversity
- Bridge between private mesh and public BGP-X network
- Providing clearnet access for mesh island users
- Enabling clearnet clients to reach mesh island services

**Multiple gateways recommended**: A mesh island should have at least 2 gateway nodes operated by different operators in different locations. Single gateway = single point of failure.

---

### Mode 6: BGP-X Range Extension — BGP-X Node v1

**DEPRECATED CONCEPT**: The earlier concept of a "BGP-X Amplifier v1" as a separate product category — a minimal LoRa repeater with no BGP-X routing intelligence — has been retired.

**Replacement**: BGP-X Node v1 in Range Extension mode.

A BGP-X Node v1 in Range Extension mode:
- Is a FULL BGP-X routing node (runs complete bgpx-node daemon)
- Prioritizes forwarding traffic over originating sessions
- Provides LoRa coverage extension
- Maintains full BGP-X onion encryption at every hop

**Why the amplifier was retired**: A pure LoRa PHY-level repeater with no BGP-X routing would relay traffic in an unencrypted intermediate state at the relay's PHY interface. This breaks end-to-end encryption. The Node v1 in Range Extension mode provides equivalent coverage while preserving all BGP-X security properties.

**Configuration for Range Extension mode**:
```toml
[node]
role = "relay"

[performance]
relay_only_mode = true
max_sessions = 500
path_construction = false

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island"
transports = ["lora"]
```

**Power**: Solar + LiFePO4 battery native. See `deployment/solar_deployment.md`.

---

## BGP and BGP-X on a Dual-Stack Router

This is the most important concept to understand: **on a dual-stack router, both BGP and BGP-X share the same WAN connection, and they don't conflict.**

### Traffic Separation

```
Packet arrives from LAN device
         │
         ▼
Routing Policy Engine evaluates:
  - Source device IP/MAC
  - Destination IP, CIDR, or domain
  - Application (process/UID in transparent proxy mode)
  - Protocol and port
  - Manual rules
         │
    ┌────┴────┐
    │ Decision │
    └────┬────┘
         │
    ┌────┴──────────────────────┐
    │                           │
    ▼                           ▼
Route via BGP-X           Route via standard BGP
(into bgpx0 TUN)          (into WAN directly)
    │                           │
    ▼                           ▼
Onion-encrypted           Plain IP packet
overlay path              standard routing
    │                           │
    ▼                           ▼
[Entry] → [Relay(s)] → [Exit]  ISP → Destination
    │
    ▼
Destination (clearnet or BGP-X native)
```

### What BGP Sees

BGP-X overlay packets appear to ISP routers as:
```
Source IP:      Router WAN IP
Destination IP: BGP-X entry node IP
Protocol:       UDP
Port:           7474
Payload:        encrypted bytes (opaque)
```

BGP cannot see inside. BGP routes these packets the same as any other UDP traffic.

### What BGP-X Sees

BGP-X uses the public internet as a packet delivery service. It knows the IP addresses of remote nodes (from DHT advertisements) and sends UDP packets to them. It has no visibility into BGP routing tables or AS-level decisions.

**Satellite WAN**: If the WAN connection is via satellite internet (Starlink, Iridium), BGP-X treats it as clearnet with satellite-class latency. No protocol change. The satellite modem is detected via USB vendor ID, and latency_class is set to satellite-leo or satellite-geo.

---

## Three Local Interfaces

The BGP-X daemon exposes three distinct local interfaces:

### Interface 1: TUN Interface (bgpx0)

A virtual network interface at the OS level. Traffic routed here is captured by the BGP-X daemon for overlay routing.

**Who uses it**: The OS routing table and iptables/nftables rules. No application connects to it directly.

**How it works**:
```
Application → OS network stack → routing table lookup →
if destination matches BGP-X rule → bgpx0 →
BGP-X daemon reads from TUN → onion encrypt →
UDP to entry node → overlay → destination
```

**Configuration**:
```
Interface:  bgpx0
MTU:        1280
Address:    100.64.0.1/10
```

### Interface 2: SDK Socket

A Unix domain socket that BGP-X native applications connect to.

**Location**: `/var/run/bgpx/sdk.sock` (Unix) or `192.168.1.1:7475` (TCP, for LAN device access to router daemon)

**Protocol**: JSON-RPC 2.0 with binary extensions for stream data

**Authentication**: Unix socket file permissions; TCP mode requires authentication token

**What it exposes**:
- Open stream to destination with path constraints
- Register as BGP-X native service
- Query which path would be used
- Manage pools for this application
- Subscribe to events (path quality, stream state, node events)
- Manage session identity (ephemeral or persistent)

**What SDK apps get back**: a local socket that looks like a normal TCP connection. All BGP-X complexity is hidden by the daemon.

**LAN device access to router's SDK socket**:

Option A (SSH forwarding):
```bash
ssh -L /tmp/bgpx-router.sock:/var/run/bgpx/sdk.sock user@router
```

Option B (TCP endpoint, configure in router daemon):
```toml
[sdk_api]
tcp_listen = "192.168.1.1:7475"
tcp_auth_required = true
```

### Interface 3: Control Socket

A Unix domain socket for the configuration client (bgpx-cli and GUI management tools).

**Location**: `/var/run/bgpx/control.sock`

**Protocol**: JSON-RPC 2.0 (text only, no binary stream data)

**Authentication**: Unix socket file permissions (must be root or bgpx group)

**What it exposes**: Full daemon management — status, statistics, path management, pool management, routing policy, node database, reputation, configuration.

**What it does NOT expose**: Stream data, application traffic, routing intelligence. It's management only.

**Satellite-connected nodes**: May have longer control command latency (600ms+ for GEO). The control protocol is designed for high-latency tolerance.

---

## HTTP/2 for BGP-X Native Applications

BGP-X native services (.bgpx addresses) use **HTTP/2 over BGP-X streams**.

HTTP/2 is selected over HTTP/3 because:
- BGP-X already provides reliable ordered delivery at the session layer
- HTTP/2's multiplexing provides stream parallelism over a single BGP-X path
- HTTP/3's QUIC would add redundant reliability and congestion control layers

HTTP/3 is used at the exit node when connecting to HTTP/3 clearnet servers — this is standard HTTP/3 over TLS over the exit's clearnet connection, not over BGP-X streams.

For LoRa paths specifically: HTTP/2 multiplexing is critical. Each round-trip on LoRa costs 1-5 seconds. HTTP/2 allows fetching multiple resources in parallel streams without additional round-trips.

---

## Application Type Model

### Type 1: Standard Applications (BGP-X Unaware)

Any application that does not use the BGP-X SDK. Browser, email client, terminal, gaming client, smart TV apps — anything.

**What they experience**: Normal network behavior. Slightly higher latency for BGP-X-routed traffic.

**What they get**: Transparent overlay protection for traffic that matches the routing policy. No special capabilities.

**What they don't get**: Path control, native addressing, service registration, event subscriptions.

**Configuration required**: None.

### Type 2: BGP-X Native Applications (SDK)

Applications built using the BGP-X SDK. Aware of the overlay and use its capabilities explicitly.

**What they can do**:
- Specify path constraints (hops, pools, ECH requirement, jurisdiction, logging policy)
- Use BGP-X native addressing (ServiceID — no IP or domain required)
- Register as BGP-X native services (receive connections through overlay without clearnet presence)
- Use double-exit architecture for maximum exit isolation
- Subscribe to path quality events and rebuild paths
- Manage session identity (ephemeral or persistent)

**How they connect**:
```
SDK App → SDK Library → Unix/TCP socket → BGP-X Daemon → Overlay Network
```

The SDK does NOT implement path construction, onion encryption, or DHT. The daemon does all of this.

**Embedded mode** (for mobile/constrained environments):
```
SDK App → SDK Library → In-Process Daemon (full stack embedded)
```

### Type 3: Configuration Client (bgpx-cli, GUIs)

The configuration client is a management tool. It is NOT a routing stack.

**Connects via**: Control Socket

**What it does**: Status, statistics, routing policy management, pool management, node database browsing, reputation viewing, configuration updates, event monitoring.

**What it does NOT do**: Route traffic, construct paths, perform handshakes, encrypt anything.

**Not required for BGP-X to function**: The daemon operates without the config client.

---

## Geographic Plausibility — OPTIONAL

Geographic plausibility scoring is an **OPTIONAL** reputation signal. Nodes MAY declare a jurisdiction in their advertisement. If they do, BGP-X measures RTT and evaluates whether the latency is plausible for the claimed region.

- If jurisdiction is NOT declared: geo plausibility scoring does NOT apply
- If jurisdiction IS declared: geo plausibility scoring applies
- Satellite-connected nodes are **exempt** from geo plausibility scoring

You are NOT required to declare your node's jurisdiction.

---

## LAN Device Experience

### Standard Device (no SDK)

1. Device connects to router WiFi or Ethernet normally
2. Gets IP from router DHCP
3. Sends packets to destinations normally
4. Router's routing policy engine classifies each flow
5. BGP-X-classified flows: silently routed through overlay
6. Standard-classified flows: routed directly to WAN
7. Device sees normal network behavior (slightly higher latency for overlay flows)

**Zero configuration required on the device.**

### SDK Application on LAN Device

The SDK application on a LAN device needs to reach the router's SDK socket:

1. Device connects to router normally
2. App uses SSH forwarding or TCP endpoint (configured on router)
3. SDK connects to router daemon
4. App calls `connect_stream("example.com:443", config)` with custom path constraints
5. Daemon builds path with specified constraints
6. SDK returns a local socket to the app
7. App reads/writes as normal TCP

The app gets explicit control over path construction that standard applications don't have.

---

## Network Diagrams by Scenario

### Scenario A: Home Network (Dual-Stack) — BGP-X Router v1

```
[Gaming console] ──── WiFi ────┐
[Work laptop]    ──── WiFi ────┤
[Phone]          ──── WiFi ────┤──► [BGP-X Router v1]
[Smart TV]       ──── Ethernet─┘         │
                                          ├── Gaming: standard route (low latency)
                                          ├── Work: BGP-X overlay (privacy)
                                          ├── Phone: BGP-X overlay (privacy)
                                          └── TV: standard route (streaming geo)
                                                    │
                                                   WAN (ISP)
```

### Scenario B: Community Mesh (Mode 4 + 5) — Node v1 + Gateway

```
[House A] ── LoRa ── [BGP-X Node v1] ── WiFi mesh ── [BGP-X Node v1 (Gateway)] ── ISP ── Internet
[House B] ── LoRa ── [BGP-X Node v1]            ↑
[House C] ── LoRa ──────────────────────────────┘
           (no ISP at houses)              (one gateway for community)

Unified DHT: Gateway stores records for mesh-only nodes
```

### Scenario C: Multi-Island via Domain Bridge

```
[Island A Mesh] ── [Domain Bridge A] ──► [Clearnet relays] ──► [Domain Bridge B] ──► [Island B Mesh]

Users on Island A communicate with users on Island B via cross-domain paths through clearnet relay pool.
Both bridge nodes participate in unified DHT.
All traffic is onion-encrypted end-to-end.
```

### Scenario D: Remote Mesh Island with Satellite Gateway

```
[Remote Island Mesh] ── LoRa ── [BGP-X Node v1 (Gateway)] ── USB ── [Starlink Terminal] ── Satellite ── Internet

Satellite WAN provides clearnet connectivity.
Latency class: satellite-leo (20-40ms RTT).
From BGP-X perspective: clearnet domain with satellite latency.
```

---

## Choosing Your Deployment

### Which Hardware?

| Need | Hardware |
|---|---|
| Replace my home router, protect all devices | BGP-X Router v1 |
| Contribute mesh coverage to my community | BGP-X Node v1 |
| High-throughput provider exit node | BGP-X Gateway v1 |
| Use my existing OpenWrt router | BGP-X OpenWrt Package |

### Dual-Stack vs BGP-X Only?

**Use Dual-Stack if**:
- Some devices need low latency (gaming, streaming) and can't tolerate 200ms overlay overhead
- You want to selectively route some traffic through the overlay and some directly
- You need some traffic to appear to come from your ISP IP (geo-restricted content)

**Use BGP-X Only if**:
- Maximum privacy enforcement is required
- You cannot risk any traffic bypassing the overlay
- All devices on the network have equivalent privacy needs

### Does My Application Need the SDK?

**No SDK needed if**:
- Your application doesn't need to control path parameters
- You don't need to host a BGP-X native service (no clearnet presence needed)
- Transparent router-level protection is sufficient

**Use SDK if**:
- You need specific path constraints (hops, pools, ECH, jurisdiction)
- You want to register a BGP-X native service (reachable without clearnet IP)
- You need path quality events and programmatic path management
- You're building a privacy-sensitive application that needs explicit overlay control

### Router vs. Standalone?

**Use Router deployment if**:
- You want to protect all devices on your network
- You want zero-configuration protection for non-technical users on the network
- You're deploying for a community or organization

**Use Standalone if**:
- You only need to protect one device
- You don't have access to the router
- You're traveling and need protection on a laptop

### Satellite WAN?

**Use satellite WAN if**:
- No fiber or cellular available at your location
- Remote community mesh island
- Mobile deployment (maritime, vehicle)

**Satellite is clearnet**: All satellite internet services are clearnet domain in BGP-X. No special configuration needed beyond latency class declaration.
