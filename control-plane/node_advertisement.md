# BGP-X Node Advertisement Specification

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

This document specifies the complete structure, validation rules, and lifecycle of BGP-X node advertisements — the primary unit of node discovery in BGP-X. This specification covers the unified advertisement format used across all routing domains.

---

## 1. Overview

A node advertisement is a signed, self-describing record published by a BGP-X node to the unified DHT to announce its existence and capabilities.

**Design Principles**:

- **Self-certifying**: The NodeID is derived from the public key; possession of the public key is sufficient to verify the NodeID without a third party.
- **Tamper-evident**: The signature covers all fields; any modification invalidates the advertisement.
- **Time-bounded**: Advertisements expire; periodic re-publication maintains freshness.
- **Honest-but-unverifiable for performance claims**: Bandwidth, latency, and uptime are self-reported; the reputation system validates these over time.
- **Domain-aware**: Advertisements describe presence in one or more routing domains (clearnet, mesh, etc.).
- **Geo Plausibility is OPTIONAL**: Jurisdiction declaration is opt-in; scoring only applies if declared.

---

## 2. Complete Advertisement Structure

The node advertisement is a JSON document. It supports both a modern `routing_domains` structure and legacy `endpoints` / `mesh_endpoints` fields for backward compatibility.

### 2.1 Full Example (Modern `routing_domains` Structure)

```json
{
  "version": 1,
  "node_id": "<32-byte BLAKE3 hash, hex>",
  "public_key": "<32-byte Ed25519 public key, base64url>",
  "roles": ["relay", "entry", "discovery"],

  "routing_domains": [
    {
      "domain_type": "clearnet",
      "domain_id": "0x00000001-00000000",
      "endpoints": [
        { "protocol": "udp", "address": "203.0.113.1", "port": 7474 },
        { "protocol": "udp", "address": "[2001:db8::1]", "port": 7474 }
      ]
    },
    {
      "domain_type": "mesh",
      "domain_id": "0x00000003-a1b2c3d4",
      "island_id": "lima-district-1",
      "mesh_transport": [
        { "transport": "wifi_mesh", "mac": "aa:bb:cc:dd:ee:ff", "mesh_id": "bgpx-lima-1" },
        { "transport": "lora", "lora_addr": "0xAABBCCDD", "frequency_mhz": 868.0, "sf": 7, "region": "EU" }
      ]
    }
  ],

  "bridge_capable": true,
  "bridges": [
    {
      "from_domain": "0x00000001-00000000",
      "to_domain": "0x00000003-a1b2c3d4",
      "bridge_latency_ms": 25,
      "bridge_transport": "wifi_mesh",
      "available": true
    }
  ],

  "exit_policy": null,
  "exit_policy_version": null,
  "bandwidth_mbps": 100,
  "latency_ms": 12,
  "uptime_pct": 99.1,
  "region": "EU",
  "country": "DE",
  "asn": 12345,
  "operator_id": "<base64url Ed25519 public key>",
  "pool_memberships": ["pool-id-hex-1"],
  "island_memberships": ["lima-district-1"],
  "pt_supported": ["obfs4"],
  "ech_capable": false,
  "is_gateway": false,
  "gateway_for_mesh": null,
  "provides_clearnet_exit": false,
  "link_quality_profiles": {
    "lora": { "latency_class": 2, "bandwidth_class": 1 }
  },
  "withdrawn": false,
  "withdrawn_at": null,
  "protocol_versions_min": 1,
  "protocol_versions_max": 1,
  "extensions": {
    "cover_traffic": true,
    "multiplexing": true,
    "path_quality_reporting": true,
    "pluggable_transport": true,
    "ech_support": false,
    "geo_plausibility": true,
    "pool_support": true,
    "mesh_transport": true,
    "advertisement_withdrawal": true,
    "session_rehandshake": true,
    "domain_bridge": true,
    "cross_domain_routing": true,
    "mesh_island_routing": true
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-04-25T12:00:00Z",
  "signature": "<64-byte Ed25519 signature, base64url>"
}
```

### 2.2 Legacy Format (Backward Compatibility)

For nodes not yet using `routing_domains`, the legacy `endpoints` and `mesh_endpoints` arrays are valid:

```json
{
  "version": 1,
  "node_id": "<32-byte BLAKE3 hash, hex>",
  "public_key": "<32-byte Ed25519 public key, base64url>",
  "roles": ["relay", "entry", "discovery"],
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 }
  ],
  "mesh_endpoints": [
    {
      "transport": "wifi_mesh",
      "mesh_id": "bgpx-community-mesh-1",
      "mac_address": "aa:bb:cc:dd:ee:ff"
    },
    {
      "transport": "lora",
      "lora_addr": "0xAABBCCDD",
      "frequency_mhz": 868.0,
      "spreading_factor": 7,
      "bandwidth_khz": 125,
      "region": "EU"
    }
  ],
  "bridge_capable": false,
  "bridges": [],
  "...": "remaining fields identical to modern format"
}
```

---

## 3. Field Specification

### 3.1 Identity Fields

#### `version` (uint, REQUIRED)

Advertisement format version. Always `1`. Receivers MUST reject advertisements with unknown versions.

#### `node_id` (string, REQUIRED)

The BLAKE3 hash of the node's Ed25519 public key, hex-encoded (64 hex characters = 32 bytes).

```
node_id = hex(BLAKE3(public_key_bytes))
```

Receivers MUST recompute this from `public_key` and verify it matches. Mismatches MUST cause rejection.

#### `public_key` (string, REQUIRED)

The node's Ed25519 public key, base64url-encoded (no padding). 32 bytes → 43 base64url characters.

#### `operator_id` (string, OPTIONAL but RECOMMENDED)

The Ed25519 public key of the node operator, base64url-encoded. Multiple nodes operated by the same entity SHOULD share an `operator_id`. This allows:
- The path selection algorithm to enforce operator diversity across a single path.
- The reputation system to aggregate behavior across all nodes from one operator.
- Clients to explicitly trust or distrust all nodes from a known operator.

If omitted, the node is treated as having a unique operator identity for diversity purposes — it will not be grouped with any other node, but also cannot benefit from cross-node reputation.

---

### 3.2 Network and Routing Domain Fields

#### `routing_domains` (array, CONDITIONAL)

The modern, preferred structure for declaring network presence. Supersedes `endpoints` and `mesh_endpoints`. If present, legacy fields are ignored. If absent, legacy fields MUST be present.

Each entry describes one routing domain where this node has active endpoints.

**Clearnet Entry**:
```json
{
  "domain_type": "clearnet",
  "domain_id": "0x00000001-00000000",
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 }
  ]
}
```
Private IP ranges (RFC 1918, loopback, link-local) MUST NOT be advertised in clearnet endpoints.

**Mesh Entry**:
```json
{
  "domain_type": "mesh",
  "domain_id": "0x00000003-a1b2c3d4",
  "island_id": "lima-district-1",
  "mesh_transport": [
    { "transport": "wifi_mesh", "mac": "aa:bb:cc:dd:ee:ff", "mesh_id": "bgpx-lima-1" },
    { "transport": "lora", "lora_addr": "0xAABBCCDD", "frequency_mhz": 868.0, "sf": 7, "region": "EU" }
  ]
}
```
`island_id` is REQUIRED for `domain_type: "mesh"`.

**LoRa Regional Entry**:
```json
{
  "domain_type": "lora-regional",
  "domain_id": "0x00000004-bb220033",
  "region_id": "eu-868-south",
  "lora_transport": {
    "frequency_mhz": 868.0,
    "spreading_factor": 9,
    "bandwidth_khz": 125,
    "lora_addr": "0xAABBCCDD"
  }
}
```

**Satellite Entry (Future/Reserved)**:
```json
{
  "domain_type": "satellite",
  "domain_id": "0x00000005-cc330044",
  "orbit_id": "bgpx-leo-constellation-alpha",
  "latency_class": 1
}
```
**Important**: Domain type `0x00000005` (bgpx-satellite) is RESERVED for future BGP-X-native satellite networks where satellites run BGP-X relay software. Commercial satellite internet services (Starlink, Iridium, Inmarsat, HughesNet, Viasat) are **clearnet domain** (`0x00000001`) with high `latency_class` annotation.

#### `endpoints` (array, CONDITIONAL)

Legacy field for internet-accessible endpoints. REQUIRED if `routing_domains` is absent. MUST be null or absent if `routing_domains` is present.

Each endpoint:
```json
{
  "protocol": "udp",
  "address": "203.0.113.1",
  "port": 7474
}
```

| Sub-field | Type | Description |
|---|---|---|
| protocol | string | Always `"udp"` for current version. |
| address | string | IPv4 or IPv6 address. MUST be public and routable. Private ranges (RFC 1918, loopback, link-local) MUST NOT be advertised. |
| port | uint16 | UDP port. Default: 7474. |

Multiple endpoints MAY be listed (e.g., one IPv4 and one IPv6 address).

#### `mesh_endpoints` (array, OPTIONAL)

Legacy field for mesh transport-specific addresses. Ignored if `routing_domains` is present.

WiFi mesh endpoint:
```json
{
  "transport": "wifi_mesh",
  "mesh_id": "bgpx-community-mesh-1",
  "mac_address": "aa:bb:cc:dd:ee:ff"
}
```

LoRa endpoint:
```json
{
  "transport": "lora",
  "lora_addr": "0xAABBCCDD",
  "frequency_mhz": 868.0,
  "spreading_factor": 7,
  "bandwidth_khz": 125,
  "region": "EU"
}
```

Bluetooth BLE endpoint:
```json
{
  "transport": "ble",
  "ble_mac": "11:22:33:44:55:66"
}
```

#### `bridge_capable` (boolean, REQUIRED)

`true` if this node has endpoints in 2+ routing domains and actively bridges traffic. When `true`, `bridges` array MUST be non-empty.

#### `bridges` (array, CONDITIONAL)

REQUIRED when `bridge_capable = true`. Describes each bridge pair this node serves.

Each entry:
```json
{
  "from_domain": "0x00000001-00000000",
  "to_domain": "0x00000003-a1b2c3d4",
  "bridge_latency_ms": 25,
  "bridge_transport": "wifi_mesh",
  "available": true
}
```

| Sub-field | Description |
|---|---|
| from_domain | Domain ID (8 bytes, hex string) of source domain. |
| to_domain | Domain ID of target domain. |
| bridge_latency_ms | Observed latency crossing this bridge. |
| bridge_transport | Primary transport used for bridging (`"wifi_mesh"`, `"lora"`, `"ble"`, `"ethernet"`). |
| available | `false` allows temporary unavailability without re-publishing entire advertisement. |

#### `asn` (uint32, REQUIRED)

ASN of the network this node connects to. Used for path diversity enforcement. Nodes MUST report their actual ASN.

#### `country` (string, REQUIRED)

ISO 3166-1 alpha-2 country code of the jurisdiction the node physically operates in. Used for geographic diversity enforcement and optional geo plausibility scoring.

Nodes MUST report the country where their physical hardware is located, not the country of the operator's headquarters.

#### `region` (string, REQUIRED)

Broad geographic region. One of:

| Code | Region |
|---|---|
| NA | North America |
| EU | Europe |
| AP | Asia-Pacific |
| SA | South America |
| AF | Africa |
| ME | Middle East |

---

### 3.3 Role Fields

#### `roles` (array of strings, REQUIRED)

The roles this node is willing to serve. At least one role MUST be present.

Valid roles:

| Role | Description |
|---|---|
| `"relay"` | Forward encrypted packets as a middle hop |
| `"entry"` | Accept client connections as the first hop |
| `"exit"` | Connect to clearnet destinations as the last hop |
| `"discovery"` | Participate in DHT and serve node lookups |

A single node MAY serve multiple roles. Nodes MUST only claim roles they are configured and capable of serving. Claiming a role the node cannot fulfill is detectable via the reputation system and results in flagging.

#### `exit_policy` (object or null, CONDITIONAL)

REQUIRED if `"exit"` is in roles. MUST be null if `"exit"` is not in roles.

Full exit policy specification:

```json
{
  "version": 1,
  "allow_protocols": ["tcp", "udp"],
  "allow_ports": [80, 443, 8080, 8443],
  "deny_destinations": [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "::1/128",
    "fc00::/7"
  ],
  "logging_policy": "none",
  "operator_contact": "ops@example.com",
  "jurisdiction": "DE",
  "dns_mode": "doh",
  "dns_resolver": "https://dns.quad9.net/dns-query",
  "dnssec_validation": true,
  "ecs_handling": "strip",
  "ech_capable": true,
  "policy_version": 1,
  "policy_previous_version": 0,
  "policy_signed_at": "2026-04-24T12:00:00Z",
  "policy_signature": "<base64url Ed25519 signature>"
}
```

| Field | Description |
|---|---|
| allow_ports | Destination TCP/UDP ports the gateway will forward. Use `[0]` to indicate all ports. |
| allow_protocols | `"tcp"`, `"udp"`, or both. |
| deny_destinations | CIDR blocks the gateway will never forward to. Private IP ranges MUST always be included to prevent LAN-targeting attacks. |
| logging_policy | `"none"` (no logs), `"metadata"` (connection timing and destination category), `"full"` (all traffic). |
| operator_contact | Public contact address for the operator. |
| jurisdiction | ISO 3166-1 alpha-2 country code of the legal jurisdiction the operator is subject to. |
| dns_mode | `"doh"`, `"dot"`, `"system"`, `"custom"`. |
| dns_resolver | URL or IP of DNS resolver (relevant for `doh`/`dot`/`custom`). |
| dnssec_validation | Whether the exit validates DNSSEC. |
| ecs_handling | `"strip"` (mandatory), `"pass"`, `"anonymize"`. |
| ech_capable | Whether this exit node supports Encrypted Client Hello (ECH). |
| policy_version | Monotonically incremented version number. |
| policy_previous_version | Previous version number for transition tracking. |
| policy_signed_at | ISO 8601 UTC timestamp of policy signing. |
| policy_signature | Ed25519 signature of the operator over canonical JSON of exit policy fields (excluding policy_signature). |

The exit policy is signed separately from the node advertisement, using the operator's key (`operator_id`). This allows the exit policy to be updated without changing the node's identity.

#### `exit_policy_version` (uint16, CONDITIONAL)

Present when `exit_policy` is present. Monotonically incremented when `exit_policy` changes. Clients detecting a version change on stream denial re-fetch the advertisement.

#### `ech_capable` (boolean, CONDITIONAL)

True if this exit node supports ECH. Requires exit role. Default: false.

#### `is_gateway` (boolean, OPTIONAL)

True if this node bridges mesh to clearnet internet.

#### `gateway_for_mesh` (array of strings, OPTIONAL)

Mesh IDs this gateway serves. Present when `is_gateway = true`.

#### `provides_clearnet_exit` (boolean, OPTIONAL)

Separate from `is_gateway`. Explicitly confirms clearnet exit capability.

---

### 3.4 Performance Fields

All performance fields are self-reported and RECOMMENDED but not REQUIRED. They are used by the path selection algorithm and validated over time by the reputation system.

#### `bandwidth_mbps` (float, RECOMMENDED)

Available bandwidth in megabits per second. Should reflect sustained throughput available for BGP-X relay traffic, not total network capacity.

#### `latency_ms` (float, RECOMMENDED)

Round-trip latency in milliseconds to the nearest major internet exchange point (IXP).

#### `uptime_pct` (float, RECOMMENDED)

Self-reported uptime percentage over the past 30 days. Range: 0.0–100.0.

---

### 3.5 Pool, Transport, and Link Quality Fields

#### `pool_memberships` (array of strings, OPTIONAL)

Pool IDs (hex) this node is currently a member of. Advisory — clients still verify individual pool membership signatures.

#### `island_memberships` (array of strings, OPTIONAL)

Mesh island IDs this node is a member of. Advisory — clients verify island DHT records independently.

#### `pt_supported` (array of strings, OPTIONAL)

Pluggable transports supported. Example: `["obfs4"]`.

#### `link_quality_profiles` (object, OPTIONAL)

Per-transport latency and bandwidth class:

```json
{
  "lora": { "latency_class": 2, "bandwidth_class": 1 },
  "satellite": { "latency_class": 3, "bandwidth_class": 3 }
}
```

Latency classes:
- 0: low (<20ms)
- 1: medium (20-200ms)
- 2: high (200ms-5s)
- 3: very-high (>5s)

Bandwidth classes:
- 0: very-low (<1Kbps)
- 1: low (1-100Kbps)
- 2: medium (100Kbps-10Mbps)
- 3: high (>10Mbps)

---

### 3.6 Withdrawal Fields

#### `withdrawn` (boolean, OPTIONAL)

If true, this node is being voluntarily withdrawn from the network. Triggers immediate removal propagation by storage nodes.

#### `withdrawn_at` (string, CONDITIONAL)

ISO 8601 timestamp. Present when `withdrawn = true`.

---

### 3.7 Protocol and Extension Fields

#### `protocol_versions_min` (uint16, REQUIRED)

The minimum protocol version this node supports.

#### `protocol_versions_max` (uint16, REQUIRED)

The maximum protocol version this node supports.

#### `extensions` (object, RECOMMENDED)

Boolean flags indicating which optional BGP-X extensions the node supports.

```json
{
  "cover_traffic": true,
  "multiplexing": true,
  "path_quality_reporting": true,
  "pluggable_transport": true,
  "ech_support": false,
  "geo_plausibility": true,
  "pool_support": true,
  "mesh_transport": true,
  "advertisement_withdrawal": true,
  "session_rehandshake": true,
  "domain_bridge": true,
  "cross_domain_routing": true,
  "mesh_island_routing": true
}
```

**Extension Flags (corresponding to bits in handshake)**:

| Extension | Description |
|---|---|
| cover_traffic | Supports generating cover traffic. |
| multiplexing | Supports stream multiplexing. |
| path_quality_reporting | Supports PATH_QUALITY_REPORT message. |
| pluggable_transport | Supports PT obfuscation layer. |
| ech_support | Supports Encrypted Client Hello at exit. |
| geo_plausibility | Participates in optional geo plausibility scoring. |
| pool_support | Supports pool-based path construction. |
| mesh_transport | Operates on mesh transports (WiFi/LoRa/BLE). |
| advertisement_withdrawal | Supports NODE_WITHDRAW message. |
| session_rehandshake | Supports 24-hour session re-handshake. |
| domain_bridge | Can act as a domain bridge node. |
| cross_domain_routing | Supports cross-domain path construction. |
| mesh_island_routing | Participates in mesh island DHT registration. |

---

### 3.8 Lifecycle Fields

#### `signed_at` (string, REQUIRED)

ISO 8601 UTC timestamp of when this advertisement was signed. Receivers MUST reject advertisements where `signed_at` is more than 60 seconds in the future (clock skew tolerance).

#### `expires_at` (string, REQUIRED)

ISO 8601 UTC timestamp after which this advertisement is no longer valid.

Rules:
- MUST be after `signed_at`.
- MUST be no more than 48 hours after `signed_at`.
- RECOMMENDED value: 24 hours after `signed_at`.

Receivers MUST reject advertisements where `expires_at` has passed.

#### `signature` (string, REQUIRED)

Ed25519 signature over the canonical JSON of all other fields (excluding `signature` itself), base64url-encoded.

---

## 4. Canonical JSON Construction

The canonical JSON used for signing is constructed as follows:

1. Take all fields of the advertisement except `signature`.
2. Sort all keys alphabetically (recursively, including nested objects).
3. Serialize to compact JSON (no whitespace, no newlines).
4. Encode as UTF-8.

Example (abbreviated):
```json
{"asn":12345,"bandwidth_mbps":100,"bridge_capable":false,"bridges":[],"country":"DE","endpoints":[...],"mesh_endpoints":[...],"node_id":"a3f2...","public_key":"...","roles":["relay","entry"],"routing_domains":[...],"signed_at":"2026-04-24T12:00:00Z","expires_at":"2026-04-25T12:00:00Z","version":1}
```

The signature is computed over the UTF-8 bytes of this canonical JSON string.

---

## 5. Validation Algorithm

Receivers MUST validate advertisements using the following algorithm before accepting them:

```python
function validate_advertisement(advert):

    # Step 1: Version check
    if advert.version != 1:
        return REJECT("Unknown advertisement version")

    # Step 2: NodeID verification
    computed_id = hex(BLAKE3(base64url_decode(advert.public_key)))
    if computed_id != advert.node_id:
        return REJECT("NodeID does not match public key")

    # Step 3: Timestamp validation
    now = current_utc_time()
    signed_at = parse_iso8601(advert.signed_at)
    expires_at = parse_iso8601(advert.expires_at)

    if signed_at > now + 60_seconds:
        return REJECT("signed_at is in the future")
    if expires_at <= now:
        return REJECT("Advertisement has expired")
    if expires_at - signed_at > 48_hours:
        return REJECT("Validity period exceeds 48 hours")

    # Step 4: Signature verification
    canonical = canonical_json(advert, exclude="signature")
    sig_bytes = base64url_decode(advert.signature)
    pub_bytes = base64url_decode(advert.public_key)
    if not ed25519_verify(pub_bytes, sig_bytes, canonical.encode("utf-8")):
        return REJECT("Signature verification failed")

    # Step 5: Role validation
    if "exit" in advert.roles and advert.exit_policy is None:
        return REJECT("Exit role requires exit_policy")
    if "exit" not in advert.roles and advert.exit_policy is not None:
        return REJECT("exit_policy present but exit role not claimed")
    if advert.ech_capable and "exit" not in advert.roles:
        return REJECT("ech_capable requires exit role")

    # Step 6: Endpoint validation (modern or legacy)
    if advert.routing_domains:
        # Modern format
        for domain in advert.routing_domains:
            if domain.domain_type not in ["clearnet", "bgpx-overlay", "mesh", "lora-regional", "satellite"]:
                return REJECT("Unknown domain_type: " + domain.domain_type)
            if domain.domain_type == "mesh" and not domain.island_id:
                return REJECT("Mesh domain requires island_id")
            if domain.domain_type == "clearnet":
                for ep in domain.endpoints:
                    if is_private_ip(ep.address):
                        return REJECT("Private IP in clearnet endpoints")
    else:
        # Legacy format
        if not advert.endpoints and not advert.mesh_endpoints:
            return REJECT("No endpoints: neither routing_domains nor legacy endpoints present")
        if advert.endpoints:
            for endpoint in advert.endpoints:
                if is_private_ip(endpoint.address):
                    return REJECT("Private IP in endpoints")

    # Step 7: Bridge validation
    if advert.bridge_capable:
        if not advert.bridges:
            return REJECT("bridge_capable=true requires non-empty bridges")
        if len(advert.routing_domains or []) < 2:
            return REJECT("bridge_capable requires endpoints in 2+ routing_domains")
        for bridge in advert.bridges:
            if not has_domain(advert.routing_domains, bridge.from_domain):
                return REJECT("Bridge from_domain not in routing_domains")
            if not has_domain(advert.routing_domains, bridge.to_domain):
                return REJECT("Bridge to_domain not in routing_domains")

    # Step 8: Withdrawal validation
    if advert.withdrawn and advert.withdrawn_at is None:
        return REJECT("withdrawn=true requires withdrawn_at")

    # Step 9: ASN validation
    if advert.asn == 0 or advert.asn > 4294967295:
        return REJECT("Invalid ASN")

    return ACCEPT
```

---

## 6. Advertisement Lifecycle

```
Generate keypair (node + optional operator)
        │
        ▼
Construct advertisement
        │
        ├── If bridge_capable: include routing_domains and bridges
        ├── If gateway: include is_gateway, gateway_for_mesh
        └── If exit: include exit_policy
        │
        ▼
Sign advertisement
        │
        ▼
Publish to unified DHT (DHT_PUT at node_advert key)
        │
        ├── Every 12 hours: re-publish node advertisement with updated timestamps
        ├── Every 8 hours: re-publish DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE
        ├── If bridge.available changes: re-publish DOMAIN_ADVERTISE immediately
        ├── If exit_policy changes: increment exit_policy_version; re-publish immediately
        ├── If pool membership changes: re-publish immediately
        └── On shutdown:
            ├── Publish NODE_WITHDRAW
            ├── Set bridge.available = false in DOMAIN_ADVERTISE and re-publish
            └── Wait up to 30s for propagation
```

**Natural Fallback**: If a node disappears without withdrawal, the advertisement expires within 48 hours. The reputation system independently tracks availability.

---

## 7. Exit Policy Change Procedure

When exit policy changes:

1. Increment `exit_policy_version` in new advertisement.
2. Set `exit_policy_previous_version` to old version number.
3. Re-sign and re-publish advertisement.
4. Old policy continues to be honored for existing streams.
5. New policy applies to new streams.
6. Clients detecting version change on stream denial re-fetch advertisement.

This allows seamless policy transitions without disrupting in-flight connections.

---

## 8. Advertisement Size Limits

Maximum advertisement size (compact JSON): **12,288 bytes** (12 KB).

This limit accommodates:
- Multiple `routing_domains` entries for multi-domain bridge nodes.
- Detailed `mesh_transport` descriptions.
- Comprehensive `link_quality_profiles`.

Advertisements exceeding 12,288 bytes MUST be rejected.

---

## 9. Geographic Plausibility — OPTIONAL Feature

Geographic plausibility scoring is an **OPTIONAL** reputation signal. It is NOT required for BGP-X operation.

### When Geo Plausibility Applies

- IF a node declares a jurisdiction (`country` field): geo plausibility scoring MAY apply.
- IF a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply.

Nodes are NOT required to declare a jurisdiction. Declaring jurisdiction is an opt-in privacy/convenience tradeoff:
- **Pro**: Users can select paths that avoid or prefer specific jurisdictions.
- **Con**: Declaring jurisdiction reveals information about the node's location.

### Exemptions

The following node types are EXEMPT from geo plausibility scoring (always return neutral score 0.5):
- Satellite-class clearnet nodes (`latency_class` indicates satellite link).
- Mesh nodes without declared jurisdiction.
- Nodes in domains without internet RTT calibration (pure mesh islands without clearnet bridge).

### Behavior in Path Construction

A node with poor geo plausibility score:
- Receives lower selection probability (scoring penalty).
- Can still be selected in a path (not hard excluded).
- Persistent implausibility may lead to reputation penalties through normal reputation system mechanisms.

Geo plausibility is a reputation signal, not a mandatory filtering criterion.

---

## 10. Operator Identity and Multi-Node Management

An operator running multiple BGP-X nodes SHOULD:

1. Generate a single operator keypair (separate from any node's keypair).
2. Include the operator's public key as `operator_id` in all managed nodes' advertisements.
3. Sign all exit policies with the operator's private key.

This enables:
- Path selection to enforce operator diversity across path segments.
- Reputation system to aggregate behavior across operator's nodes.
- Pool curators to accept/reject operators as a group.

Operator private key MUST be stored securely, separately from node private keys. A compromised node private key does not compromise the operator identity. A compromised operator private key can be revoked by re-publishing all affected nodes' advertisements with a new `operator_id` and signing a revocation notice.
