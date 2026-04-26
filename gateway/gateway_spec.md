# BGP-X Gateway and Domain Bridge Node Specification

**Version**: 0.1.0-draft

---

## 1. Generalized Role: Domain Bridge Node

The "gateway" concept is generalized to the **Domain Bridge Node** — any node with endpoints in two or more routing domains.

The previous notion of "gateway" (mesh ↔ internet bridge with clearnet exit) is a specific type of domain bridge node. All gateway properties are retained as instances of the general domain bridge model.

**Domain bridge node types**:

| Type | From Domain | To Domain | Exit Policy Required |
|---|---|---|---|
| Clearnet exit gateway | clearnet | clearnet (different network) | Yes |
| Mesh gateway (with exit) | clearnet | mesh:\<island_id\> | Yes |
| Mesh relay bridge (no exit) | clearnet | mesh:\<island_id\> | No |
| Mesh-to-mesh bridge | mesh:\<A\> | mesh:\<B\> | No |
| Satellite gateway | clearnet | satellite:\<orbit_id\> | Yes |
| Satellite-mesh bridge | satellite:\<X\> | mesh:\<Y\> | Yes |

All types use the same daemon code and DOMAIN_BRIDGE hop type. The distinction is configuration.

---

## 2. Architecture

```
Any domain (source)
         │
         ▼
Onion decryption (this node's layer — hop_type = 0x06 DOMAIN_BRIDGE)
         │
         ├── Exit policy check (ONLY when target destination = clearnet internet)
         │   → DENY → STREAM_CLOSE (ERR_EXIT_POLICY_DENIED)
         │   → ALLOW ↓
         │
         ▼
Cross-domain path_id registration
  path_id → source_domain, source_addr
  path_id → target_domain, target_addr (from subsequent hops)
         │
         ▼
Target domain transport selection
         │
         ▼
Target domain address resolution
         │
         ▼
Forward remaining onion payload via target transport (NO re-encryption)

Return traffic (from target domain):
  path_id cross-domain lookup → source_domain, source_addr
  Forward opaque blob via source domain transport (NO decryption)
```

---

## 3. Exit Policy Enforcement

Exit policy applies ONLY when:
- The bridge connects to clearnet AND
- The traffic is destined for the public internet (not for another BGP-X node or mesh service)

For mesh-to-mesh bridges: no exit policy needed.
For clearnet-to-mesh bridges (relay only): no exit policy needed.
For clearnet exit gateways: exit policy required as previously specified.

---

## 4. DoH DNS Resolver (Clearnet Exit Only)

Required only for nodes with clearnet exit capability. Unchanged specification.

```toml
[exit_policy]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
```

ECS stripping (MANDATORY for exit nodes): strip EDNS Client Subnet from DNS queries to prevent client IP subnet leakage to DNS resolvers and destination servers.

---

## 5. ECH Support (Clearnet Exit Only)

ECH applies only when the bridge includes a clearnet exit. Unchanged specification. Mesh relay bridge nodes do not perform TLS ClientHello and do not need ECH.

---

## 6. Clearnet Client → Mesh Island Service

The most important capability enabled by the domain bridge model. A clearnet client with no mesh hardware can reach a mesh island service:

```
Clearnet client (no mesh radio, no BGP-X router)
         │ UDP/IP
         ▼
[Clearnet relay nodes — standard onion encryption]
         │ UDP/IP
         ▼
[Domain Bridge Node — clearnet UDP endpoint + mesh radio]
  1. Receives onion packet via clearnet UDP
  2. Decrypts its DOMAIN_BRIDGE layer
  3. Stores path_id → clearnet predecessor (for return routing)
  4. Forwards remaining payload via mesh radio
         │ WiFi mesh or LoRa
         ▼
[Mesh relay nodes — onion encryption in mesh domain]
         │ mesh transport
         ▼
[Mesh service]
```

Return path:
```
Mesh service → mesh relays → bridge node (via mesh radio, opaque)
Bridge node: cross-domain path_id lookup → clearnet predecessor
Bridge node → clearnet relays → client (via UDP, opaque)
```

The clearnet client needs no mesh hardware, no BGP-X router, no special configuration. The bridge node handles all domain translation. The client's clearnet IP is hidden from the mesh service.

---

## 7. Mesh-to-Mesh Bridge

A node with two mesh transport interfaces bridges two mesh islands directly without clearnet:

```
Mesh Island A (WiFi mesh)
         │
[Multi-transport Bridge Node]
  Receives from Island A via WiFi mesh
  Decrypts DOMAIN_BRIDGE layer
  Forwards to Island B via LoRa
         │
Mesh Island B (LoRa)
```

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "island-a"
transports = ["wifi_mesh"]
wifi_mesh_interface = "mesh0"

[[routing_domains]]
domain_type = "mesh"
island_id = "island-b"
transports = ["lora"]
lora_interface = "/dev/ttyUSB0"
lora_frequency_mhz = 868.0

[domain_bridge]
enabled = true
```

No clearnet exit policy needed. No DoH DNS needed. Pure radio-to-radio domain bridge.

---

## 8. Unified DHT (Replaces Two-DHT Model)

Gateway nodes NO LONGER synchronize between two separate DHTs. Instead:

- When online: gateway publishes mesh island records directly to the unified internet DHT
- Mesh-only nodes' advertisements are stored in unified DHT by the gateway on their behalf
- When gateway goes offline: records remain valid until TTL; new records cannot be published
- When gateway reconnects: any pending record updates published automatically

No synchronization algorithm needed — there is only one DHT.

---

## 9. Response Handling and Return Path

Return traffic from the clearnet destination travels back through the SAME path in reverse using path_id routing:

1. Response arrives at gateway from clearnet destination
2. Gateway encrypts response using K_exit (session key shared with client)
3. Gateway looks up path_id in cross-domain path table → last clearnet relay
4. Sends encrypted blob via clearnet UDP to last clearnet relay (opaque)
5. Each clearnet relay forwards opaquely via path_id
6. Client decrypts with K_exit

For mesh service responses, the mesh service encrypts with K_service and return path crosses domain boundary at bridge node as described in ARCHITECTURE.md Section 4.2.

---

## 10. Exit Policy Versioning (Unchanged)

When exit policy changes: increment `exit_policy_version`, set `exit_policy_previous_version`, re-publish advertisement. Old policy honored for existing streams; new policy for new streams. Domain bridge nodes without clearnet exit functionality do not use exit policy versioning.

---

## 11. Logging Policy (Unchanged)

`logging_policy = "none"` (technical enforcement): no file handles for access logging; aggregate stats in memory only; only daemon error events logged.

---

## 12. Monitoring

All previous exit node metrics retained. New:

```
bgpx_domain_bridge_transitions_total    # Packets forwarded across domain boundary
bgpx_domain_bridge_latency_ms          # Average bridge transition latency
bgpx_cross_domain_return_forwarded     # Return packets forwarded cross-domain
bgpx_mesh_radio_send_total             # Packets sent via mesh transport
bgpx_mesh_radio_recv_total             # Packets received via mesh transport
bgpx_cross_domain_path_table_size      # Active entries in cross-domain path table
```
