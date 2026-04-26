# BGP-X Protocol Versioning Policy

**Version**: 0.1.0-draft

---

## 1. Purpose

This document defines how the BGP-X protocol evolves, how versions are negotiated, and what compatibility guarantees exist between versions.

---

## 2. Version Format

```
MAJOR.MINOR
```

Wire encoding: single uint16 = `(MAJOR << 8) | MINOR`

Current: v0.1 = `0x0001`

All features in the v1 specification are included in version 0.1. Nothing is deferred. This includes the complete cross-domain routing model, N-hop unlimited policy, unified DHT, and all new message types.

---

## 3. Version Negotiation

Version negotiation occurs during handshake. Domain-agnostic — identical negotiation regardless of routing domain.

**HANDSHAKE_INIT**: client sends supported version list (array of uint16, most preferred first).

**HANDSHAKE_RESP**: node sends selected version = highest version in intersection of node-supported and client-supported.

If intersection is empty: HANDSHAKE_RESP with ERR_PROTOCOL_VERSION.

Version is locked for the session after handshake. No mid-session version switching.

---

## 4. Extension Flags

Extension capabilities negotiated independently of protocol version via extension flags in handshake.

| Bit | Extension | v0.1 |
|---|---|---|
| 0 | Cover traffic support | Yes |
| 1 | Pluggable transport support | Yes |
| 2 | Stream multiplexing | Yes |
| 3 | Path quality reporting (with domain ID) | Yes |
| 4 | ECH support (exit nodes only) | Yes |
| 5 | Geographic plausibility scoring | Yes |
| 6 | Pool support (including domain-scoped pools) | Yes |
| 7 | Mesh transport support | Yes |
| 8 | Advertisement withdrawal | Yes |
| 9 | Session re-handshake (24hr) | Yes |
| 10 | Domain bridge support | Yes |
| 11 | Cross-domain path construction | Yes |
| 12 | Mesh island routing | Yes |
| 13-31 | Reserved | No |

**Bit 10 — Domain bridge support**: node can act as a domain bridge (forward between routing domains). Nodes declaring bit 10 MUST have valid bridge pair records in DHT and MUST handle DOMAIN_BRIDGE (0x06), MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09) hop types. Nodes without bit 10 can still relay cross-domain traffic (they just forward RELAY packets without interpreting DOMAIN_BRIDGE).

**Bit 11 — Cross-domain path construction**: node's path selection algorithm supports `domain_segments` construction. Not required for relaying.

**Bit 12 — Mesh island routing**: node participates in mesh island DHT registration and island advertisement propagation.

**Backward compatibility**: nodes not declaring bits 10-12 can still participate in the clearnet-only overlay exactly as before. Cross-domain functionality requires the new extension bits only for nodes that actively bridge or construct cross-domain paths.

---

## 5. Compatibility Rules

### MINOR Version Changes

- New message types may be added (unknown types gracefully ignored by older implementations)
- New fields may be added to existing message types (appended; safe defaults when absent)
- Existing message type behavior is preserved
- Implementations MUST accept messages with unknown trailing bytes in payloads

### MAJOR Version Changes

- Explicitly breaking; not backward compatible
- Old implementations not required to support new MAJOR version
- Transition period: previous MAJOR supported for minimum 12 months
- Bootstrap nodes: MUST support current MAJOR and immediately preceding MAJOR

### Pool and Domain Record Versioning

Pool advertisement format version field = 1. Domain bridge advertisement version = 1. Mesh island advertisement version = 1. New DHT record types can be added without protocol version increment.

---

## 6. Pre-Release Versions (MAJOR = 0)

Pre-release versions (v0.x) have NO backward compatibility guarantees. MAY break between MINOR increments. Intended for development and testing. Minimum 60-day public review period before 1.0 release.

---

## 7. Deprecation Process

1. Announcement in CHANGELOG and documentation
2. Warning period (6 months): client software warns when connecting to deprecated versions
3. End of support: removed from default supported_versions
4. Removal: deprecated version removed from official software

---

## 8. Version Advertisement in Node Records

```json
{
  "protocol_versions_min": 1,
  "protocol_versions_max": 1
}
```

For cross-domain capability:
```json
{
  "extensions": {
    "domain_bridge": true,
    "cross_domain_routing": true,
    "mesh_island_routing": true
  }
}
```

Clients pre-filter nodes by version compatibility before initiating handshakes.
