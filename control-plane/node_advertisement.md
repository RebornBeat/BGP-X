# BGP-X Node Advertisement Specification

**Version**: 0.1.0-draft

This document specifies the complete structure, validation rules, and lifecycle of BGP-X node advertisements.

---

## 1. Overview

A node advertisement is a signed, self-describing record that a BGP-X node publishes to the DHT to announce its existence and capabilities. Advertisements are the primary mechanism by which clients discover available relay, entry, and exit nodes.

The advertisement is designed to be:

- **Self-certifying**: The NodeID is derived from the public key, so possession of the public key is sufficient to verify the NodeID without a third party
- **Tamper-evident**: The signature covers all fields; any modification invalidates the advertisement
- **Time-bounded**: Advertisements expire and must be periodically re-published to prevent stale data
- **Honest-but-unverifiable for performance claims**: Bandwidth, latency, and uptime are self-reported; the reputation system validates these over time

---

## 2. Advertisement Structure

Node advertisements are JSON documents. The canonical form (used for signing) is compact JSON with keys sorted alphabetically.

### 2.1 Base Advertisement

```json
{
  "asn": 12345,
  "bandwidth_mbps": 100,
  "country": "DE",
  "endpoints": [
    {
      "address": "203.0.113.1",
      "port": 7474,
      "protocol": "udp"
    }
  ],
  "exit_policy": null,
  "expires_at": "2026-04-25T12:00:00Z",
  "extensions": {
    "cover_traffic": true,
    "multiplexing": true,
    "path_quality_reporting": true,
    "pluggable_transport": false
  },
  "latency_ms": 12,
  "node_id": "a3f2...b9c1",
  "operator_id": "d7e4...2a8f",
  "protocol_versions_max": 1,
  "protocol_versions_min": 1,
  "public_key": "base64url_encoded_ed25519_public_key",
  "region": "EU",
  "roles": ["relay", "entry"],
  "signed_at": "2026-04-24T12:00:00Z",
  "signature": "base64url_encoded_ed25519_signature",
  "uptime_pct": 99.1,
  "version": 1
}
```

---

## 3. Field Specification

### 3.1 Identity Fields

#### `version` (uint, REQUIRED)

Advertisement format version. Currently `1`. Receivers MUST reject advertisements with unknown versions.

#### `node_id` (string, REQUIRED)

The BLAKE3 hash of the node's Ed25519 public key, hex-encoded (64 hex characters = 32 bytes).

```
node_id = hex(BLAKE3(public_key_bytes))
```

Receivers MUST recompute this from `public_key` and verify it matches. Mismatches MUST cause rejection.

#### `public_key` (string, REQUIRED)

The node's Ed25519 public key, base64url-encoded (no padding). 32 bytes → 43 base64url characters.

#### `operator_id` (string, RECOMMENDED)

The Ed25519 public key of the node operator, base64url-encoded. Multiple nodes operated by the same entity SHOULD share an operator_id. This allows the reputation system to track operator-level behavior and allows the path selection algorithm to enforce operator diversity.

If omitted, the node is treated as having a unique operator identity for diversity purposes — it will not be grouped with any other node, but also cannot benefit from cross-node reputation.

---

### 3.2 Network Fields

#### `endpoints` (array, REQUIRED)

Array of network endpoints where the node accepts BGP-X connections. At least one endpoint MUST be present.

Each endpoint:

```json
{
  "address": "203.0.113.1",
  "port": 7474,
  "protocol": "udp"
}
```

| Sub-field | Type | Description |
|---|---|---|
| address | string | IPv4 or IPv6 address. MUST be a public, routable address. Private address ranges (RFC 1918, RFC 4193) MUST NOT be advertised. |
| port | uint16 | UDP port. Default: 7474. |
| protocol | string | Always "udp" for the current version. |

Multiple endpoints MAY be listed (e.g., one IPv4 and one IPv6 address).

#### `asn` (uint32, REQUIRED)

The Autonomous System Number of the network the node is connected to. Used for path diversity enforcement. Nodes MUST report their actual ASN.

Clients verify ASN consistency over time via the reputation system. A node that frequently changes its reported ASN without a corresponding change in IP routing is flagged.

#### `country` (string, REQUIRED)

ISO 3166-1 alpha-2 country code (e.g., "US", "DE", "JP") of the jurisdiction the node physically operates in. Used for geographic diversity enforcement.

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

A single node MAY serve multiple roles. Typical configurations:

- Small relay: `["relay", "discovery"]`
- Full relay: `["relay", "entry", "discovery"]`
- Gateway: `["exit", "discovery"]`
- Combined: `["relay", "entry", "exit", "discovery"]`

Nodes MUST only claim roles they are configured and capable of serving. Claiming a role the node cannot fulfill is detectable via the reputation system and results in flagging.

#### `exit_policy` (object or null, CONDITIONAL)

REQUIRED if `"exit"` is in roles. MUST be null if `"exit"` is not in roles.

Full exit policy specification:

```json
{
  "allow_ports": [80, 443, 8080, 8443],
  "allow_protocols": ["tcp", "udp"],
  "deny_destinations": [
    "10.0.0.0/8",
    "172.16.0.0/12",
    "192.168.0.0/16",
    "127.0.0.0/8",
    "::1/128",
    "fc00::/7"
  ],
  "jurisdiction": "DE",
  "logging_policy": "none",
  "operator_contact": "ops@example.com",
  "policy_signature": "base64url_encoded_ed25519_signature",
  "policy_signed_at": "2026-04-24T12:00:00Z",
  "version": 1
}
```

| Field | Description |
|---|---|
| allow_ports | Destination TCP/UDP ports the gateway will forward. Use `[0]` to indicate all ports. |
| allow_protocols | `"tcp"`, `"udp"`, or both. |
| deny_destinations | CIDR blocks the gateway will never forward to. Private IP ranges MUST always be included to prevent LAN-targeting attacks. |
| jurisdiction | ISO 3166-1 alpha-2 country code of the legal jurisdiction the operator is subject to. |
| logging_policy | `"none"` (no logs), `"metadata"` (connection timing and destination category), `"full"` (all traffic). |
| operator_contact | Public contact address for the operator. |
| policy_signature | Ed25519 signature of the operator over canonical JSON of exit policy fields (excluding policy_signature). |
| policy_signed_at | Timestamp of policy signing. |
| version | Exit policy format version. Currently 1. |

The exit policy is signed separately from the node advertisement, using the operator's key (operator_id). This allows the exit policy to be updated without changing the node's identity.

---

### 3.4 Performance Fields

All performance fields are self-reported and RECOMMENDED but not REQUIRED. They are used by the path selection algorithm and validated over time by the reputation system.

#### `bandwidth_mbps` (float, RECOMMENDED)

Available bandwidth in megabits per second. This should reflect the sustained throughput available for BGP-X relay traffic, not the node's total network capacity.

#### `latency_ms` (float, RECOMMENDED)

Round-trip latency in milliseconds to the nearest major internet exchange point (IXP). This gives clients a rough estimate of the hop's latency contribution.

#### `uptime_pct` (float, RECOMMENDED)

Self-reported uptime percentage over the past 30 days. Range: 0.0–100.0.

---

### 3.5 Protocol Fields

#### `protocol_versions_min` (uint16, REQUIRED)

The minimum protocol version this node supports (encoded as a uint16; see `/protocol/versioning.md`).

#### `protocol_versions_max` (uint16, REQUIRED)

The maximum protocol version this node supports.

#### `extensions` (object, RECOMMENDED)

Boolean flags indicating which optional BGP-X extensions the node supports.

```json
{
  "cover_traffic": true,
  "multiplexing": true,
  "path_quality_reporting": false,
  "pluggable_transport": false
}
```

---

### 3.6 Lifecycle Fields

#### `signed_at` (string, REQUIRED)

ISO 8601 UTC timestamp of when this advertisement was signed. Example: `"2026-04-24T12:00:00Z"`.

Receivers MUST reject advertisements where `signed_at` is more than 60 seconds in the future (clock skew tolerance).

#### `expires_at` (string, REQUIRED)

ISO 8601 UTC timestamp after which this advertisement is no longer valid.

Rules:
- MUST be after `signed_at`
- MUST be no more than 48 hours after `signed_at`
- RECOMMENDED value: 24 hours after `signed_at`

Receivers MUST reject advertisements where `expires_at` has passed.

#### `signature` (string, REQUIRED)

Ed25519 signature over the canonical JSON of all other fields (excluding `signature` itself), base64url-encoded.

---

## 4. Canonical JSON Construction

The canonical JSON used for signing is constructed as follows:

1. Take all fields of the advertisement except `signature`
2. Sort all keys alphabetically (recursively, including nested objects)
3. Serialize to compact JSON (no whitespace, no newlines)
4. Encode as UTF-8

Example (abbreviated):

```json
{"asn":12345,"bandwidth_mbps":100,"country":"DE","endpoints":[...],...,"version":1}
```

The signature is computed over the UTF-8 bytes of this canonical JSON string.

---

## 5. Validation Algorithm

Receivers MUST validate advertisements using the following algorithm before accepting them:

```
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

    # Step 5: Role-specific validation
    if "exit" in advert.roles and advert.exit_policy is None:
        return REJECT("Exit role requires exit_policy")

    if "exit" not in advert.roles and advert.exit_policy is not None:
        return REJECT("exit_policy present but exit role not claimed")

    # Step 6: Endpoint validation
    if len(advert.endpoints) == 0:
        return REJECT("No endpoints")

    for endpoint in advert.endpoints:
        if is_private_ip(endpoint.address):
            return REJECT("Private IP in endpoints")

    # Step 7: ASN validation
    if advert.asn == 0 or advert.asn > 4294967295:
        return REJECT("Invalid ASN")

    return ACCEPT
```

---

## 6. Advertisement Lifecycle

```
Node generates keypair
        │
        ▼
Node constructs advertisement
        │
        ▼
Node signs advertisement
        │
        ▼
Node publishes to DHT (DHT_PUT)
        │
        ▼
        ├── Every 12 hours: re-publish with updated timestamps
        │
        ├── If configuration changes: re-publish immediately
        │
        └── If node shuts down: advertisement expires naturally
                    (no explicit withdrawal mechanism)
```

There is no explicit withdrawal message. When a node goes offline, its advertisement expires within 48 hours. The reputation system independently tracks node availability and marks unavailable nodes before their advertisements expire.

---

## 7. Operator Identity and Multi-Node Management

An operator running multiple BGP-X nodes SHOULD:

1. Generate a single operator keypair (separate from any node's keypair)
2. Include the operator's public key as `operator_id` in all managed nodes' advertisements
3. Sign all exit policies with the operator's private key

This allows:
- The path selection algorithm to enforce operator diversity across a single path
- The reputation system to aggregate behavior across all nodes from one operator
- Clients to explicitly trust or distrust all nodes from a known operator

The operator private key MUST be stored securely and separately from node private keys. A compromised node private key does not compromise the operator identity. A compromised operator private key can be revoked by re-publishing all affected nodes' advertisements with a new operator_id and signing a revocation notice.

---

## 8. Advertisement Size Limits

Maximum advertisement size: **4096 bytes** (compact JSON)

This limit ensures:
- Advertisements fit in a single BGP-X packet
- DHT storage per record is bounded
- Bulk advertisement parsing is efficient

Advertisements exceeding 4096 bytes MUST be rejected.
