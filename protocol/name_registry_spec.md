# BGP-X Name Registry Specification

**Version**: 0.1.0-draft

---

## 1. Overview

The BGP-X Name Registry provides human-readable short names for `.bgpx` services. It maps `shortname.bgpx` addresses to ServiceIDs via a dedicated DHT layer.

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
    "icon_hash": "blake3_hex_of_icon_png"
  },
  "signature": "ed25519_hex_signature_of_canonical_fields"
}
```

**Signature scope** (what is signed):

```
canonical = "name_registry_v1:" || name || ":" || service_id || ":" || registered_at_decimal || ":" || expires_at_decimal
signature = Ed25519Sign(service_private_key, BLAKE3(canonical))
```

The signature is produced by the service's Ed25519 private key (the key whose public key IS the ServiceID). This proves the name was registered by the service operator, not an impostor.

**TTL**: `expires_at` is a Unix timestamp. DHT TTL = `expires_at - current_time`. Maximum registration period: 90 days. Standard renewal period: every 30 days.

---

## 4. DHT Key Derivation

Name records are stored in the Name Registry DHT (separate from but using the same Kademlia algorithm as the routing DHT):

```
dht_key = BLAKE3("name_registry_v1:" || normalize(name))

normalize(name) = lowercase(trim(name))
```

The normalization ensures `Community-News`, `community-news`, and `COMMUNITY-NEWS` all resolve to the same key.

---

## 5. Name Registration Process

### 5.1 Via bgpx-cli

```bash
# Register a name for a local service
bgpx-cli names register community-news --service a3f2b9c1...

# This generates a NameRecord signed with the service's key
# and publishes it to the Name Registry DHT
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

First-registered-wins. The existing registration cannot be displaced by a different ServiceID while it is valid.

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

**Renewal record**: same format as registration; updated `registered_at` and `expires_at` fields; new signature. Published to DHT, replaces existing record.

---

## 7. Name Withdrawal

An operator can explicitly withdraw their name registration before expiry:

```bash
bgpx-cli names withdraw community-news --service a3f2b9c1...

# Withdrawal record:
# { "version": 1, "name": "community-news", "withdrawn_at": ..., "signature": ... }
# Published to DHT; existing record replaced; name immediately available for re-registration
```

**Withdrawal record** differs from registration record: `withdrawn_at` field instead of `expires_at`; DHT TTL = 1 hour (short; just long enough for propagation). After withdrawal propagates, the name is available for registration by any ServiceID.

---

## 8. Reserved Names

The following names are reserved and cannot be registered:

```
bgpx, bgpx-network, bgpx-org, bootstrap, bootstrap-node,
dht, relay, gateway, bridge, exit, node, router,
admin, administration, network, security, privacy,
error, offline, warning, alert, system, root
```

Reserved names return `BGPX_NAME_RESERVED` error on registration attempt.

---

## 9. Name Discovery

### Lookup by Name

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

### List Own Names

```bash
bgpx-cli names list --self
# Output:
# community-news.bgpx    → a3f2... (expires 2026-04-15)
# radio-station.bgpx     → b7c2... (expires 2026-03-01, RENEWAL DUE SOON)
```

### Directory Search

BGP-X Browser new tab page displays a browsable directory of registered `.bgpx` services. This directory is populated from publicly accessible Name Registry DHT records (operators opt-in to directory listing via `metadata.public_directory: true` in their name record).

---

## 10. Security Properties

### What the Name Registry Does NOT Provide

- **No uniqueness guarantee beyond first-registered-wins**: if `news.bgpx` is registered, `news.bgpx` by a different ServiceID cannot be registered while valid. But `news-network.bgpx` or similar names are uncontrolled.
- **No trademark protection**: registering a name does not grant trademark rights. BGP-X is not responsible for name disputes.
- **No identity verification**: anyone can register any name for any ServiceID. The name registry proves ownership-by-key, not legitimacy of the service.
- **No squatting prevention**: operators may register names defensively. No mechanism prevents this.

### What the Name Registry DOES Provide

- **Proof of service control**: the signature on a NameRecord proves that the ServiceID's private key holder registered this name. An impostor cannot register a name for a ServiceID they don't control.
- **Tamper detection**: DHT record manipulation is detectable via signature verification. Any modification to a NameRecord invalidates the signature.
- **Availability decentralization**: the Name Registry DHT has no central server to take down. Names remain resolvable as long as any DHT node holds the record.

### Phishing via Name Registry

Users should verify the ServiceID matches what they expect. A malicious operator could register `community-news.bgpx` for a ServiceID that is NOT the legitimate community news service. The BGP-X Browser displays the ServiceID alongside the name; users should verify the ServiceID matches known-good values for critical services.

High-security services should publish their canonical ServiceID via out-of-band channels (physical flyers, trusted third-party directories, social verification) so users can verify.

---

## 11. Name Registry DHT Implementation Notes

The Name Registry DHT uses the same Kademlia algorithm as the routing DHT but is a **logically separate DHT instance** on the same nodes.

**Key space**: same 256-bit key space as routing DHT.
**Nodes**: any BGP-X node participates in both DHTs simultaneously.
**Bootstrap**: same bootstrap process as routing DHT; new nodes join both DHTs via bootstrap contacts.
**Record validation**: nodes validate NameRecord signatures before storing; invalid records rejected.
**Rate limiting**: nodes limit NameRecord PUTs from a single source to prevent spam (max 10 name registrations per hour per source IP in routing DHT sense).

The two DHTs are differentiated by key prefix in the BGP-X daemon:

```rust
// Routing DHT key (node advertisement):
let routing_key = blake3::hash(b"node_adv_v1:" + node_id.as_bytes());

// Name Registry DHT key:
let name_key = blake3::hash(b"name_registry_v1:" + name.as_bytes());
```

The prefix prevents key collisions between routing DHT records and Name Registry records in the same DHT storage layer.
