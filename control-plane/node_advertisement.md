# BGP-X Node Advertisement Specification

**Version**: 0.1.0-draft

---

## 1. Overview

A node advertisement is a signed, self-describing record published by a BGP-X node to the unified DHT. All routing domains use the same advertisement format.

---

## 2. Complete Advertisement Structure

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
    }
  ],

  "bridge_capable": false,
  "bridges": [],

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
  "island_memberships": [],
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
    "mesh_transport": false,
    "advertisement_withdrawal": true,
    "session_rehandshake": true,
    "domain_bridge": false,
    "cross_domain_routing": true,
    "mesh_island_routing": false
  },
  "signed_at": "2026-04-24T12:00:00Z",
  "expires_at": "2026-04-25T12:00:00Z",
  "signature": "<64-byte Ed25519 signature, base64url>"
}
```

---

## 3. Field Specifications

### `routing_domains` (array, REQUIRED)

Supersedes legacy `endpoints` and `mesh_endpoints` fields. Each entry describes one routing domain where this node has active endpoints.

**Clearnet entry**:
```json
{
  "domain_type": "clearnet",
  "domain_id": "0x00000001-00000000",
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.1", "port": 7474 }
  ]
}
```

Private IP ranges MUST NOT be advertised in clearnet endpoints.

**Mesh entry**:
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

**LoRa regional entry**:
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

**Satellite entry**:
```json
{
  "domain_type": "satellite",
  "domain_id": "0x00000005-cc330044",
  "orbit_id": "starlink-leo",
  "wan_provider": "starlink",
  "latency_class": 2
}
```

### `bridge_capable` (boolean, REQUIRED)

`true` if this node has endpoints in 2+ routing domains and actively bridges traffic. When `true`, `bridges` array MUST be non-empty.

### `bridges` (array, CONDITIONAL)

Required when `bridge_capable = true`. Each entry:

```json
{
  "from_domain": "0x00000001-00000000",
  "to_domain": "0x00000003-a1b2c3d4",
  "bridge_latency_ms": 25,
  "bridge_transport": "wifi_mesh",
  "available": true
}
```

`available = false` allows a bridge-capable node to temporarily mark a bridge as unavailable (e.g., mesh radio offline for maintenance) without full advertisement re-publication.

### `island_memberships` (array, OPTIONAL)

Mesh island IDs this node is a member of. Advisory — clients verify island DHT records independently.

### New Extension Flags

```json
"extensions": {
  "domain_bridge": true,          // Can bridge routing domains (handles 0x06-0x09 hop types)
  "cross_domain_routing": true,   // Supports cross-domain path construction requests
  "mesh_island_routing": true     // Participates in mesh island DHT registration
}
```

### `exit_policy` (updated with new fields)

```json
{
  "version": 1,
  "allow_protocols": ["tcp", "udp"],
  "allow_ports": [80, 443, 8080, 8443],
  "deny_destinations": ["10.0.0.0/8", "192.168.0.0/16", "172.16.0.0/12", "127.0.0.0/8"],
  "logging_policy": "none",
  "operator_contact": "operator@example.com",
  "jurisdiction": "DE",
  "dns_mode": "doh",
  "dns_resolver": "https://dns.quad9.net/dns-query",
  "dnssec_validation": true,
  "ecs_handling": "strip",
  "ech_capable": true,
  "policy_version": 1,
  "policy_previous_version": 0,
  "policy_signed_at": "2026-04-24T12:00:00Z",
  "policy_signature": "<Ed25519 signature by operator_id key>"
}
```

---

## 4. Validation Algorithm

```python
function validate_advertisement(advert):

    if advert.version != 1:
        return REJECT("Unknown version")

    computed_id = hex(BLAKE3(base64url_decode(advert.public_key)))
    if computed_id != advert.node_id:
        return REJECT("NodeID mismatch")

    now = current_utc_time()
    signed_at = parse_iso8601(advert.signed_at)
    expires_at = parse_iso8601(advert.expires_at)

    if signed_at > now + 60:
        return REJECT("signed_at in future")
    if expires_at <= now:
        return REJECT("Expired")
    if expires_at - signed_at > 48 * 3600:
        return REJECT("Validity period too long")

    canonical = canonical_json(advert, exclude="signature")
    sig_bytes = base64url_decode(advert.signature)
    pub_bytes = base64url_decode(advert.public_key)
    if not ed25519_verify(pub_bytes, sig_bytes, canonical.encode("utf-8")):
        return REJECT("Signature invalid")

    if "exit" in advert.roles and advert.exit_policy is None:
        return REJECT("Exit role requires exit_policy")
    if "exit" not in advert.roles and advert.exit_policy is not None:
        return REJECT("exit_policy present without exit role")
    if advert.ech_capable and "exit" not in advert.roles:
        return REJECT("ech_capable requires exit role")

    if advert.routing_domains:
        for domain in advert.routing_domains:
            if domain.domain_type not in ["clearnet", "bgpx-overlay", "mesh", "lora-regional", "satellite"]:
                return REJECT("Unknown domain_type")
            if domain.domain_type == "mesh" and not domain.island_id:
                return REJECT("Mesh domain requires island_id")
            if domain.domain_type == "clearnet":
                for ep in domain.endpoints:
                    if is_private_ip(ep.address):
                        return REJECT("Private IP in clearnet endpoints")

    if advert.bridge_capable:
        if not advert.bridges:
            return REJECT("bridge_capable=true requires non-empty bridges")
        if len(advert.routing_domains) < 2:
            return REJECT("bridge_capable requires endpoints in 2+ routing_domains")
        for bridge in advert.bridges:
            if not has_domain(advert.routing_domains, bridge.from_domain):
                return REJECT("Bridge from_domain not in routing_domains")
            if not has_domain(advert.routing_domains, bridge.to_domain):
                return REJECT("Bridge to_domain not in routing_domains")

    if not advert.routing_domains and not advert.endpoints:
        return REJECT("No endpoints: neither routing_domains nor legacy endpoints present")

    if advert.withdrawn and advert.withdrawn_at is None:
        return REJECT("withdrawn=true requires withdrawn_at")

    return ACCEPT
```

---

## 5. Advertisement Lifecycle

```
Generate keypair
     │
     ▼
Construct advertisement (including routing_domains and bridges if applicable)
     │
     ▼
Sign advertisement
     │
     ▼
Publish to unified DHT (DHT_PUT at node_advert key)
If bridge_capable: also publish DOMAIN_ADVERTISE at each bridge pair key
If is_gateway/island_manager: also publish MESH_ISLAND_ADVERTISE at island key
     │
     ├── Every 12 hours: re-publish node advertisement with updated timestamps
     ├── Every 8 hours: re-publish DOMAIN_ADVERTISE and MESH_ISLAND_ADVERTISE
     ├── If bridge.available changes: re-publish DOMAIN_ADVERTISE immediately
     ├── If exit_policy changes: increment exit_policy_version; re-publish immediately
     └── On shutdown:
         Publish NODE_WITHDRAW
         Set bridge.available = false in DOMAIN_ADVERTISE and re-publish
         Wait up to 30s for propagation
```

---

## 6. Advertisement Size Limits

Maximum advertisement size (compact JSON): **12,288 bytes** (12 KB — increased to accommodate multiple routing domain entries for multi-domain bridge nodes).

Advertisements exceeding 12,288 bytes MUST be rejected.
