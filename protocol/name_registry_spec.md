# BGP-X Name Registry Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X Name Registry provides human-readable short names for `.bgpx` services. It maps `shortname.bgpx` addresses to ServiceIDs via a dedicated key space within the BGP-X Unified DHT.

The Name Registry is:
- **Decentralized**: no central authority; names stored in DHT
- **Self-authenticating**: name records signed by the service's own Ed25519 key
- **First-registered-wins**: no registration fees, no bureaucracy
- **Expiring**: names must be renewed every 30 days or they lapse

---

## 2. Name Format Rules

Valid `.bgpx` short names:

```
[a-z0-9-]{3,63}

Rules:
- Lowercase alphanumeric and hyphens only
- Minimum 3 characters
- Maximum 63 characters
- Cannot start or end with a hyphen
- Cannot contain consecutive hyphens (--) unless explicitly allowed (for community names)
- Cannot be a reserved name (see Section 8)
```

Valid examples: `community-news`, `library`, `radio-station-lima`, `k2ndm`

Invalid examples: `-test` (leading hyphen), `test-` (trailing hyphen), `AB` (too short, uppercase), `my name` (space not allowed)

### 2.1 Relationship to Full `.bgpx` Addresses

BGP-X supports two addressing formats for services:

1.  **Full `.bgpx` address**: `service_id_hex.bgpx` (64 hex characters, self-authenticating).
    *   Format: `a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx`
    *   This is the canonical, unforgeable address derived directly from the service's Ed25519 public key.
    *   See `/protocol/bgpx_domain_spec.md` for the full address specification, including cross-domain addressing formats like `service_id.mesh.island_id.bgpx`.

2.  **Short name**: `shortname.bgpx` (via Name Registry).
    *   Format: `community-news.bgpx`
    *   This is a human-readable alias resolved via this specification.

Short names map to full ServiceIDs. A client resolving `community-news.bgpx` obtains the 32-byte ServiceID, which is then used for path construction and endpoint discovery via the routing DHT.

---

## 3. Name Record Format

```json
{
  "version": 1,
  "name": "community-news",
  "service_id": "a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1",
  "registered_at": 1737000000,
  "expires_at": 1739592000,
  "metadata": {
    "description": "Community news service for Lima District 1 mesh island",
    "service_type": "web",
    "icon_hash": "blake3_hex_of_icon_png",
    "public_directory": true
  },
  "signature": "ed25519_hex_signature_of_canonical_fields"
}
```

**Field Descriptions:**

| Field | Type | Description |
|---|---|---|
| `version` | uint | Record format version. Current: `1`. |
| `name` | string | The short name (without `.bgpx` suffix). |
| `service_id` | string (hex) | The 32-byte Ed25519 public key (ServiceID) this name maps to. |
| `registered_at` | uint64 | Unix timestamp of initial registration. |
| `expires_at` | uint64 | Unix timestamp when the record expires. |
| `metadata` | object | Optional descriptive fields. |
| `metadata.description` | string | Human-readable description of the service. |
| `metadata.service_type` | string | Type hint (e.g., `web`, `api`, `message`). |
| `metadata.icon_hash` | string | BLAKE3 hash of an icon image for directory display. |
| `metadata.public_directory` | boolean | If `true`, this service appears in the BGP-X Browser directory. Default: `false`. |
| `signature` | string (hex) | Ed25519 signature proving ownership. |

### 3.1 Signature Scope

The signature is produced by the **service's Ed25519 private key** (the key whose public key IS the ServiceID). This proves the name was registered by the service operator, not an impostor.

**Canonical Signing Input:**

```
canonical = "name_registry_v1:" || name || ":" || service_id || ":" || registered_at_decimal || ":" || expires_at_decimal
digest = BLAKE3(canonical)
signature = Ed25519Sign(service_private_key, digest)
```

### 3.2 TTL and Renewal

- **Maximum registration period**: 90 days.
- **Standard renewal period**: Every 30 days.
- **DHT TTL**: `expires_at - current_time`.

---

## 4. DHT Key Derivation

Name records are stored in the **BGP-X Unified DHT** (the same DHT infrastructure used for routing advertisements, pool records, and domain bridges).

To prevent key collisions, a specific prefix is used:

```
dht_key = BLAKE3("name_registry_v1:" || normalize(name))

normalize(name) = lowercase(trim(name))
```

**Key Space Separation**: The `name_registry_v1:` prefix ensures name records do not collide with node advertisements (`node_adv_v1:`), pool records, or domain bridge records in the unified DHT key space.

**Normalization**: `Community-News`, `community-news`, and `COMMUNITY-NEWS` all resolve to the same key.

---

## 5. Name Registration Process

### 5.1 Via bgpx-cli

```bash
# Register a name for a local service
bgpx-cli names register community-news --service a3f2b9c1...

# Output:
# Name registered: community-news.bgpx
# ServiceID: a3f2b9c1...
# Expires: 2026-05-15T12:00:00Z
# Renewal due: 2026-04-15T12:00:00Z (30 days before expiry)
```

### 5.2 Via SDK

```rust
let name_registry = client.name_registry().await?;
let record = name_registry.register(
    "community-news",
    service_keypair,
    NameMetadata {
        description: "Community news for Lima District 1".to_string(),
        service_type: ServiceType::Web,
        icon_hash: None,
        public_directory: true,
    },
    Duration::from_days(90),
).await?;
println!("Registered: {}.bgpx → {}", record.name, record.service_id.to_hex());
```

### 5.3 Collision Handling

If a name is already registered by another service:

```
bgpx-cli names register community-news --service <different_service_id>

Error: Name 'community-news.bgpx' is already registered
       Registered by: b4e9... (different ServiceID)
       Expires: 2026-04-01T00:00:00Z
       If this is your service, use --service b4e9... to renew it.
       Otherwise, choose a different name.
```

**First-registered-wins.** The existing registration cannot be displaced by a different ServiceID while it is valid.

**After expiry**: if the existing registration has expired (past `expires_at`), a new registration by any ServiceID is accepted.

---

## 6. Name Renewal

Names must be renewed before `expires_at` or they lapse and become available for re-registration.

```bash
# Manual renewal
bgpx-cli names renew community-news --service a3f2b9c1...

# Automatic renewal (daemon handles this for registered services)
# Daemon checks registered names every 24 hours
# Renews automatically 7 days before expiry
```

**Renewal record**: Same format as registration; updated `registered_at` and `expires_at` fields; new signature. Published to DHT, replaces existing record.

---

## 7. Name Withdrawal

An operator can explicitly withdraw their name registration before expiry:

```bash
bgpx-cli names withdraw community-news --service a3f2b9c1...

# Withdrawal record:
# { "version": 1, "name": "community-news", "withdrawn_at": ..., "signature": ... }
# Published to DHT; existing record replaced; name immediately available for re-registration
```

**Withdrawal record** differs from registration record:
- Contains `withdrawn_at` field instead of `expires_at`.
- DHT TTL = 1 hour (short; just long enough for propagation).
- After withdrawal propagates, the name is available for registration by any ServiceID.

---

## 8. Reserved Names

The following names are reserved and cannot be registered:

```
bgpx, bgpx-network, bgpx-org, bootstrap, bootstrap-node,
dht, relay, gateway, bridge, exit, node, router,
admin, administration, network, security, privacy,
error, offline, warning, alert, system, root,
api, www, mail, smtp, imap, pop, ftp,
localhost, test, example, invalid, local
```

Reserved names return `BGPX_NAME_RESERVED` error on registration attempt.

---

## 9. Name Discovery

### 9.1 Lookup by Name

```bash
bgpx-cli names lookup community-news
# Output:
# Name: community-news.bgpx
# ServiceID: a3f2b9c1d4...
# Description: Community news service for Lima District 1
# Registered: 2026-01-15T12:00:00Z
# Expires: 2026-04-15T12:00:00Z
# Status: valid
```

### 9.2 Resolution Workflow

When a client (e.g., BGP-X Browser) navigates to `community-news.bgpx`:

1.  **Name Resolution**: Client queries the Unified DHT for `BLAKE3("name_registry_v1:community-news")`.
2.  **Record Validation**: Client verifies the NameRecord signature using the embedded public key.
3.  **Service Lookup**: Client extracts the `service_id` from the NameRecord.
4.  **Endpoint Discovery**: Client queries the routing DHT for `BLAKE3("node_adv_v1:" || service_id)` (or uses a DHT query for the ServiceID directly) to find the service's advertised endpoints (clearnet, mesh island, etc.).
5.  **Path Construction**: Client constructs a path to the discovered endpoint using standard BGP-X path construction (see `/protocol/path_construction.md`).

### 9.3 List Own Names

```bash
bgpx-cli names list --self
# Output:
# community-news.bgpx    → a3f2... (expires 2026-04-15)
# radio-station.bgpx     → b7c2... (expires 2026-03-01, RENEWAL DUE SOON)
```

### 9.4 Directory Search

BGP-X Browser new tab page displays a browsable directory of registered `.bgpx` services. This directory is populated from Name Registry DHT records where `metadata.public_directory: true`.

---

## 10. Security Properties

### 10.1 What the Name Registry Does NOT Provide

- **No uniqueness guarantee beyond first-registered-wins**: If `news.bgpx` is registered, `news.bgpx` by a different ServiceID cannot be registered while valid. But `news-network.bgpx` or similar names are uncontrolled.
- **No trademark protection**: Registering a name does not grant trademark rights. BGP-X is not responsible for name disputes.
- **No identity verification**: Anyone can register any name for any ServiceID. The name registry proves ownership-by-key, not legitimacy of the service.
- **No squatting prevention**: Operators may register names defensively. No mechanism prevents this.
- **No jurisdiction enforcement**: The Name Registry is global. It does not enforce jurisdictional restrictions on names.

### 10.2 What the Name Registry DOES Provide

- **Proof of service control**: The signature on a NameRecord proves that the ServiceID's private key holder registered this name. An impostor cannot register a name for a ServiceID they don't control.
- **Tamper detection**: DHT record manipulation is detectable via signature verification. Any modification to a NameRecord invalidates the signature.
- **Availability decentralization**: The Name Registry uses the Unified DHT, which has no central server to take down. Names remain resolvable as long as any DHT node holds the record.

### 10.3 Phishing via Name Registry

Users should verify the ServiceID matches what they expect. A malicious operator could register `community-news.bgpx` for a ServiceID that is NOT the legitimate community news service.

**Mitigation**:
- The BGP-X Browser displays the ServiceID alongside the name.
- Users should verify the ServiceID matches known-good values for critical services (published via out-of-band channels: physical flyers, trusted directories, social verification).

---

## 11. Name Registry DHT Implementation Notes

The Name Registry uses the **BGP-X Unified DHT**. It is not a physically separate network, but a logically separated key space.

**Nodes**: Any BGP-X node participating in the Unified DHT automatically participates in the Name Registry (stores name records if they fall in the node's key range).

**Bootstrap**: Same bootstrap process as the routing DHT.

**Record Validation**: Nodes MUST validate NameRecord signatures before storing. Invalid records MUST be rejected with a DHT error.

**Rate Limiting**: Nodes SHOULD limit NameRecord PUTs from a single source to prevent spam (e.g., max 10 name registrations per hour per source IP). This is a local policy, not a protocol rule.

**Key Prefixes** (for implementation reference):

```rust
// Routing DHT key (node advertisement):
let routing_key = blake3::hash(b"node_adv_v1:" + node_id.as_bytes());

// Name Registry DHT key:
let name_key = blake3::hash(b"name_registry_v1:" + name.as_bytes());
```

The distinct prefixes ensure no collision between different record types in the unified key space.

---

## 12. Integration with Cross-Domain Services

The ServiceID resolved from a short name may belong to a service in **any routing domain**:
- Clearnet-exit service
- Mesh island service (e.g., `mesh:lima-district-1`)
- Hybrid service with endpoints in multiple domains

The Name Registry does not store domain information. The resolved ServiceID's advertisement in the routing DHT provides the endpoint details and domain context. This design keeps the Name Registry simple while allowing it to address services in all BGP-X routing domains transparently.
