# BGP-X Gateway and Domain Bridge Node Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X gateway (exit node) and the generalized domain bridge node — the components bridging the BGP-X overlay to the public internet and between routing domains.

---

## 1. Role and Responsibility

### 1.1 Generalized Role: Domain Bridge Node

The "gateway" concept is generalized to the **Domain Bridge Node** — any node with endpoints in two or more routing domains. The previous notion of "gateway" (mesh ↔ internet bridge with clearnet exit) is a specific type of domain bridge node. All gateway properties are retained as instances of the general domain bridge model.

A BGP-X gateway is the translation layer between:

```
BGP-X Overlay (private, encrypted, identity-based)
                        ↕
        Gateway (translation + exit policy enforcement)
                        ↕
  Public Internet (IP-addressed, standard BGP routing)
```

**Domain bridge node types**:

| Type | From Domain | To Domain | Exit Policy Required |
|---|---|---|---|
| Clearnet exit gateway | clearnet | clearnet (different network) | Yes |
| Mesh gateway (with exit) | clearnet | mesh:\<island_id\> | Yes |
| Mesh relay bridge (no exit) | clearnet | mesh:\<island_id\> | No |
| Mesh-to-mesh bridge | mesh:\<A\> | mesh:\<B\> | No |
| Satellite gateway | clearnet | satellite:\<orbit_id\> | Yes |
| Satellite-mesh bridge | satellite:\<X\> | mesh:\<Y\> | Yes |

All types use the same daemon code and DOMAIN_BRIDGE hop type (0x06). The distinction is configuration.

### 1.2 Security Significance

The gateway is the most security-sensitive component because:
- It is the only node that sees destination addresses (when acting as exit)
- It is the point at which BGP-X traffic enters the public internet
- Its behavior is the most legally consequential for its operator

All gateway behavior must be transparent, policy-driven, and auditable.

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

For clearnet exit:
BGP-X overlay side                              Clearnet side
─────────────────────────────────────────────────────────────
BGP-X packet received from overlay
         │
         ▼
Onion decryption (final layer)
         │
         ▼
Exit policy engine ──── DENY ──────► STREAM_CLOSE (ERR_EXIT_POLICY_DENIED)
         │
        ALLOW
         │
         ▼
Stream demultiplexer
         │
         ├── TCP streams ──► DoH DNS resolver ──► ECH manager ──► TCP connection ──► Internet
         │
         └── UDP streams ──► UDP socket manager ──────────────────────────────────► Internet
                                          │
                                          ▼
                                 Response handler
                                          │
                                 Encrypt with K_exit
                                          │
                                 path_id routing → return path
```

---

## 3. Domain Bridge Node Operations

### 3.1 Cross-Domain Path Routing

When a domain bridge node receives a DOMAIN_BRIDGE hop (0x06):

1. Decrypt its own onion layer.
2. Extract target domain ID (bytes 0-7 of `next_hop` field).
3. Extract target bridge node ID (bytes 8-39 of `next_hop` field).
4. Store `path_id → source_addr` in the current domain's path table.
5. Store `path_id → {source_domain, source_addr, target_domain}` in the cross-domain path table.
6. Select transport appropriate to the target domain.
7. Forward the remaining onion payload via the target-domain transport to the bridge node.
8. No re-encryption: the remaining payload is already encrypted for subsequent hops.

### 3.2 Mesh-to-Mesh Bridge Configuration

A node with two mesh transport interfaces bridges two mesh islands directly without clearnet:

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

## 4. Exit Gateway Operations

### 4.1 Exit Policy Enforcement

Exit policy applies ONLY when:
- The bridge connects to clearnet AND
- The traffic is destined for the public internet (not for another BGP-X node or mesh service)

For mesh-to-mesh bridges: no exit policy needed.
For clearnet-to-mesh bridges (relay only): no exit policy needed.
For clearnet exit gateways: exit policy required as specified below.

### 4.2 Enforcement Algorithm

```python
function evaluate_stream_open(stream_open):

    protocol = determine_protocol(stream_open.flags)
    destination = parse_destination(stream_open.destination)
    port = stream_open.port

    # Protocol check
    if protocol not in exit_policy.allow_protocols:
        return DENY(ERR_EXIT_POLICY_DENIED)

    # Port check
    if exit_policy.allow_ports != [0]:
        if port not in exit_policy.allow_ports:
            return DENY(ERR_EXIT_POLICY_DENIED)

    # DNS resolution (for domain destinations)
    if destination.type == DOMAIN:
        resolved_ip = resolve_via_doh(destination.domain)
        if resolved_ip is None:
            return DENY(ERR_DESTINATION_UNREACHABLE)
        destination.ip = resolved_ip

    # Private IP check
    if is_private_ip(destination.ip):
        return DENY(ERR_EXIT_POLICY_DENIED)

    # Deny destination check
    for cidr in exit_policy.deny_destinations:
        if destination.ip in cidr:
            return DENY(ERR_EXIT_POLICY_DENIED)

    # ECH check
    if stream_open.flags.ECH_REQUIRED and not exit_policy.ech_capable:
        return DENY(ERR_ECH_REQUIRED_NOT_SUPPORTED)

    return ALLOW(resolved_ip=destination.ip)
```

### 4.3 Policy Enforcement Logging

When a stream is denied: log denial category (port/protocol/destination class), NOT the full destination address.

When a stream is allowed with `logging_policy = "none"`: log NOTHING.

---

## 5. DoH DNS Resolver

Exit gateways MUST use encrypted DNS:

```toml
[exit_policy]
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
```

**dns_mode options**:
- `doh`: DNS-over-HTTPS (RECOMMENDED)
- `dot`: DNS-over-TLS
- `system`: system resolver (NOT recommended for exit nodes)
- `custom`: custom resolver with full URL

**ECS stripping (MANDATORY for exit nodes)**:

The DNS resolver MUST strip EDNS Client Subnet (ECS) extension from queries. ECS leaks the client's IP address subnet to DNS resolvers and destination servers.

**DNSSEC validation**: validates DNSSEC signatures to prevent DNS spoofing. Configurable; strongly recommended.

---

## 6. ECH Support

### 6.1 ECH Negotiation Procedure

When a TLS stream is being opened to a clearnet destination:

```
1. Query DNS (DoH) for HTTPS record of destination domain
2. If HTTPS record contains ECHConfig:
   a. Use ECH in TLS ClientHello
   b. Inner ClientHello: real SNI (encrypted to server ECH key)
   c. Outer ClientHello: generic SNI (".")
   d. Set ECH_NEGOTIATED=1, ECH_AVAILABLE=1 in first response DATA flags

3. If no ECHConfig:
   a. If ECH_REQUIRED in stream flags:
      → STREAM_CLOSE with ERR_ECH_REQUIRED_NOT_SUPPORTED
   b. If ECH_PREFERRED only:
      → Proceed with standard TLS
      → Set ECH_NEGOTIATED=0, ECH_AVAILABLE=0 in first response DATA flags

4. If ECH negotiation fails (after successful config fetch):
   a. One retry without ECH if ECH_REQUIRED=false
   b. If ECH_REQUIRED=true: STREAM_CLOSE with ERR_ECH_NEGOTIATION_FAILED
```

### 6.2 ECH Config Cache

- TTL: 1 hour per domain
- Cleared on daemon restart (conservative: re-fetch always)
- `bgpx-cli exit ech-cache` shows current cached configs
- `bgpx-cli exit clear-ech-cache` forces re-fetch

### 6.3 ECH and Privacy

ECH hides the domain name from:
- The exit node itself (encrypted in inner ClientHello)
- Network observers between exit node and destination
- ISP of the destination server

ECH does NOT hide:
- The destination IP address (BGP routes to it in plaintext)
- That a TLS connection was made
- The server's certificate public key

---

## 7. Connection Management

### 7.1 TCP Connection Pool

```
connection_pool: HashMap<DestinationKey, ConnectionPool>
DestinationKey = (resolved_ip, port)

Per pool:
  max_connections: 50 per destination
  idle_timeout: 60 seconds
  connect_timeout: 10 seconds
```

### 7.2 Connection Limits

| Limit | Default |
|---|---|
| max_outbound_connections | 2,000 total |
| max_connections_per_destination | 50 |
| connection_timeout | 10 seconds |
| idle_connection_timeout | 60 seconds |

### 7.3 UDP Handling

One ephemeral UDP socket per BGP-X stream. 5-minute inactivity timeout.

---

## 8. Response Handling and Return Path

Responses travel back to the client via path_id routing:

```
1. Response arrives at gateway from clearnet destination
2. Gateway encrypts response using K_exit (session key shared with client)
3. Gateway looks up path_id in cross-domain path table → last clearnet relay
4. Sends encrypted blob via clearnet UDP to last clearnet relay (opaque)
5. Each clearnet relay forwards opaquely via path_id
6. Client decrypts with K_exit
```

Exit node does NOT construct full onion layers for return traffic. It encrypts once with K_exit. Intermediate relays are opaque forwarders.

For mesh service responses, the mesh service encrypts with K_service and return path crosses domain boundary at bridge node.

### 8.1 Response Buffering

TCP response buffering:
- Buffer size: 64 KB per stream
- Flush trigger: buffer full, FIN received, or 10ms timeout
- Maximum response packet size: 1160 bytes (BGP-X MTU)

Large responses automatically segmented into multiple BGP-X packets.

---

## 9. Logging Policy

### 9.1 `logging_policy = "none"` (RECOMMENDED)

Technical enforcement (not just policy):
- No file handles opened for access logging
- Aggregate stats in memory only (cleared on restart)
- No connection records written anywhere
- Only daemon error events logged (startup, shutdown, configuration errors)

### 9.2 Prohibited Log Items

Gateway nodes MUST NOT log:
- Client IPs
- Destinations
- path_id values
- Path composition
- Session IDs
- Traffic volume per path
- Pool query history
- Cross-domain traversal details per session
- Mesh island identifiers associated with specific sessions

### 9.3 `logging_policy = "metadata"`

Logged per connection:
- Timestamp
- Protocol (TCP/UDP)
- Destination port
- Destination IP hash (not plaintext)
- Bytes transferred
- Duration

NOT logged: full destination IP/domain, content, BGP-X session info.

### 9.4 `logging_policy = "full"`

Full access log including destination IP, port, bytes, timing. Intended only for legally mandated logging. Clients SHOULD avoid full-logging gateways for sensitive traffic.

---

## 10. Exit Policy Versioning

When exit policy changes:

1. Increment `exit_policy_version` in advertisement
2. Set `exit_policy_previous_version` to old version
3. Re-publish advertisement
4. Old policy honored for existing streams (clients see old version from cached advertisement)
5. New policy applies to new streams (clients with new version see new policy)

Client behavior on stream denial from gateway:
- If denial is ERR_EXIT_POLICY_DENIED and advertisement has new policy_version
- Client re-fetches advertisement, evaluates new policy
- Retries stream open if acceptable under new policy, or fails if not

---

## 11. Pool Participation

Exit nodes may be members of multiple pools:

```toml
[advertisement]
pool_memberships = ["pool-id-1", "pool-id-2"]  # Announced to clients
```

Pool-specific exit policies: v1 uses single exit policy for all pools. Future: per-pool policies.

---

## 12. Mesh Gateway Operation

When configured as a gateway (bridge between mesh and internet):

```toml
[advertisement]
is_gateway = true
gateway_for_mesh = ["bgpx-community-mesh-1"]
provides_clearnet_exit = true
```

### 12.1 Unified DHT Participation

Gateway nodes do NOT synchronize between two separate DHTs. Instead, they participate in the **unified DHT**:

- When online: gateway publishes mesh island records directly to the unified internet DHT
- Mesh-only nodes' advertisements are stored in unified DHT by the gateway on their behalf
- When gateway goes offline: records remain valid until TTL; new records cannot be published
- When gateway reconnects: any pending record updates published automatically

No synchronization algorithm needed — there is only one DHT.

---

## 13. Satellite WAN

When the gateway uses satellite WAN (Starlink, etc.):

```json
{
  "link_quality_profiles": {
    "satellite": { "latency_class": 2, "bandwidth_class": 3 }
  }
}
```

This signals to path selection that this gateway has high-latency WAN. Interactive traffic paths avoid satellite gateways; bulk transfer paths may prefer them.

KEEPALIVE timeout extended for satellite-WAN gateways (configurable up to 5 minutes).

**Important**: Commercial satellite services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) provide **internet connectivity over BGP**. From BGP-X's perspective, they are clearnet transports — the same clearnet domain as fiber or cellular, just with higher latency. They are not a separate mesh routing domain. See `docs/satellite_architecture.md`.

---

## 14. Monitoring Metrics

Gateway nodes MUST expose the following Prometheus metrics:

### 14.1 Exit Node Metrics

| Metric Name | Type | Description |
|---|---|---|
| `bgpx_exit_connections_total` | Counter | Total outbound connections established |
| `bgpx_exit_policy_denials_total` | Counter | Denials by category (label: `reason`) |
| `bgpx_exit_dns_resolution_latency_ms` | Gauge | DNS resolution time |
| `bgpx_exit_dns_failures_total` | Counter | DNS resolution failures |
| `bgpx_ech_connections_total` | Counter | Connections using ECH |
| `bgpx_ech_fallback_total` | Counter | Connections where ECH fell back |
| `bgpx_ech_failure_total` | Counter | ECH negotiation failures |
| `bgpx_active_outbound_connections` | Gauge | Current active clearnet connections |

### 14.2 Domain Bridge Metrics

| Metric Name | Type | Description |
|---|---|---|
| `bgpx_domain_bridge_transitions_total` | Counter | Packets forwarded across domain boundary |
| `bgpx_domain_bridge_latency_ms` | Gauge | Average bridge transition latency |
| `bgpx_cross_domain_return_forwarded` | Counter | Return packets forwarded cross-domain |
| `bgpx_cross_domain_path_table_size` | Gauge | Active entries in cross-domain path table |

### 14.3 Mesh Radio Metrics

| Metric Name | Type | Description |
|---|---|---|
| `bgpx_mesh_radio_send_total` | Counter | Packets sent via mesh transport |
| `bgpx_mesh_radio_recv_total` | Counter | Packets received via mesh transport |

### 14.4 Gateway DHT Metrics

| Metric Name | Type | Description |
|---|---|---|
| `bgpx_gateway_dht_stores_for_mesh_total` | Counter | Records stored on behalf of mesh-only nodes |

---

## 15. Configuration Reference

```toml
[gateway]
role = "exit"                    # "exit", "domain_bridge", or "both"
logging_policy = "none"           # "none", "metadata", "full"

[exit_policy]
version = 1
allow_protocols = ["tcp", "udp"]
allow_ports = [80, 443, 8080, 8443]
deny_destinations = ["10.0.0.0/8", "192.168.0.0/16", "172.16.0.0/12"]
logging_policy = "none"
operator_contact = "operator@example.com"
jurisdiction = "DE"
dns_mode = "doh"
dns_resolver = "https://dns.quad9.net/dns-query"
dnssec_validation = true
ecs_handling = "strip"
ech_capable = true
policy_signed_at = "2026-04-24T12:00:00Z"
policy_signature = "<Ed25519 signature by operator_id key>"

[domain_bridge]
enabled = true
bridges = [
  {
    from_domain = "0x00000001-00000000",
    to_domain = "0x00000003-a1b2c3d4",
    bridge_latency_ms = 25,
    available = true
  }
]
```
