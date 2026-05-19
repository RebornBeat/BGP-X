# BGP-X Protocol Versioning Policy

**Version**: 0.1.0-draft

---

## 1. Purpose

This document defines how the BGP-X protocol evolves over time, how versions are negotiated, and what compatibility guarantees are made between versions.

---

## 2. Version Format

BGP-X uses a two-component version number:

```
MAJOR.MINOR
```

- **MAJOR**: Incremented for breaking changes that are incompatible with previous versions
- **MINOR**: Incremented for backward-compatible additions and clarifications

The protocol version is a single uint16 on the wire:

```
wire_version = (MAJOR << 8) | MINOR
```

For example, version 1.3 is encoded as `0x0103`.

The current specification version is `0.1` (`0x0001` on the wire, during pre-release).

All features in the v1 specification are included in version 0.1. This specification does not defer any features to future versions. This includes the complete cross-domain routing model, N-hop unlimited policy, unified DHT, and all new message types.

---

## 3. Version Negotiation

Version negotiation occurs during the handshake. The handshake protocol is **domain-agnostic** — identical negotiation regardless of routing domain (clearnet, mesh, satellite-connected clearnet).

### 3.1 Client behavior

HANDSHAKE_INIT includes a **supported version list** — an array of uint16 version numbers the client supports, from most preferred to least preferred.

```
supported_versions = [0x0001, 0x0002]  # client supports v0.1 and v0.2
```

### 3.2 Node behavior

HANDSHAKE_RESP includes a **selected version** — the highest version number that appears in both the node's supported versions and the client's supported versions.

```
selected_version = max(intersection(node_supported, client_supported))
```

If the intersection is empty (no common version), the node sends HANDSHAKE_RESP with error code ERR_PROTOCOL_VERSION and closes the session.

### 3.3 Version lock

Once a version is selected during the handshake, all subsequent messages in that session MUST use the selected version. Version switching mid-session is not permitted.

---

## 4. Extension Flags

Extension capabilities are negotiated independently of protocol version via the extension flags in HANDSHAKE_INIT and HANDSHAKE_RESP. This allows extensions to be deployed without version increments.

### v0.1 Extension Flag Table

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

### Extension Flag Details

**Bit 0 — Cover traffic support**: Node can generate and process COVER packets (0x11). COVER packets use the **same session_key as RELAY packets** — there is no separate cover_key.

**Bit 1 — Pluggable transport support**: Node supports obfuscation layer (obfs4-style or external PT subprocess). Applies below BGP-X protocol.

**Bit 2 — Stream multiplexing**: Node supports multiplexing multiple logical streams over a single path.

**Bit 3 — Path quality reporting**: Node generates PATH_QUALITY_REPORT (0x16) messages with domain ID field. Nodes without this extension can still relay PQR opaquely.

**Bit 4 — ECH support**: For exit nodes only. Node can negotiate Encrypted Client Hello with TLS 1.3 destinations that publish ECH configuration. Requires DoH DNS resolver.

**Bit 5 — Geographic plausibility scoring**: Node participates in RTT-based geographic validation. **OPTIONAL** — applies only if jurisdiction is declared in node advertisement.

**Bit 6 — Pool support**: Node can participate in pool-based routing. Pool-aware paths (segments from different pools) require bit 6 to be accepted by all hops. Implementations that do not support pools cannot participate in multi-segment paths.

**Bit 7 — Mesh transport support**: Node operates on mesh transports (WiFi 802.11s, LoRa, BLE). Mesh transport requires bit 7 to be accepted. Internet-only nodes declare bit 7 = 0.

**Bit 8 — Advertisement withdrawal**: Node supports NODE_WITHDRAW (0x15) message processing and propagation.

**Bit 9 — Session re-handshake**: Node supports 24-hour session key rotation via re-handshake.

**Bit 10 — Domain bridge support**: Node can act as a domain bridge (forward between routing domains). Nodes declaring bit 10 MUST:
- Have valid bridge pair records in DHT
- Handle DOMAIN_BRIDGE (0x06), MESH_ENTRY (0x07), MESH_EXIT (0x08), MESH_RELAY (0x09) hop types
- Maintain cross-domain path_id routing table

Nodes without bit 10 can still relay cross-domain traffic — they forward RELAY packets without interpreting DOMAIN_BRIDGE layers.

**Bit 11 — Cross-domain path construction**: Node's path selection algorithm supports `domain_segments` construction. Not required for relaying; required for nodes that construct cross-domain paths.

**Bit 12 — Mesh island routing**: Node participates in mesh island DHT registration and island advertisement propagation.

### Backward Compatibility

Nodes not declaring bits 10-12 can still participate in the clearnet-only overlay exactly as before. Cross-domain functionality requires the new extension bits only for nodes that actively bridge or construct cross-domain paths.

---

## 5. Compatibility Rules

### 5.1 MINOR version changes

MINOR version increments are backward-compatible. A node running version 1.3 can communicate with a node running version 1.1:

- New message types added in 1.3 are unknown to 1.1 nodes but MUST be gracefully ignored by 1.1 implementations
- New fields added to existing message types MUST be appended (not inserted) and MUST have safe defaults when absent
- The behavior of all existing message types is preserved

Implementations MUST accept messages with unknown trailing bytes in payloads and MUST NOT reject them.

### 5.2 MAJOR version changes

MAJOR version increments are explicitly breaking. Implementations that support MAJOR version N are not required to support MAJOR version N-1. However:

- Implementations SHOULD support at least one previous MAJOR version for a transition period of no less than 12 months
- A deprecation notice MUST be published no less than 6 months before a MAJOR version becomes the minimum required version
- Bootstrap nodes distributed with client software MUST support the current MAJOR version and the immediately preceding MAJOR version

### 5.3 Extension negotiation independence

Extensions (cover traffic, pluggable transport, mesh transport, domain bridge, etc.) are negotiated via the extension flags in HANDSHAKE_INIT and HANDSHAKE_RESP, independently of the protocol version. This allows extensions to be deployed without version increments.

A client and relay that both support an extension activate it; a relay that doesn't support an extension degrades gracefully.

---

## 6. Independent Format Versioning

Several data structures are versioned independently of the protocol version:

### 6.1 Pool Advertisement Format

Pool advertisement JSON includes a `"version"` field. Current: `1`. Pool member list format is versioned independently within the pool advertisement.

### 6.2 Pool Curator Key Rotation Records

Key rotation records include a `"version"` field. Current: `1`. See `/protocol/pool_curator_key_rotation.md`.

### 6.3 Domain Bridge Advertisement Format

Domain bridge records (DOMAIN_ADVERTISE, 0x1A) include a `"version"` field. Current: `1`.

### 6.4 Mesh Island Advertisement Format

Mesh island records (MESH_ISLAND_ADVERTISE, 0x1B) include a `"version"` field. Current: `1`.

### 6.5 Mesh Transport Protocol

Mesh transport protocol is versioned as a transport-type extension. New transport types can be added without protocol version increment.

### 6.6 Node Advertisement Format

Node advertisement JSON includes a `"version"` field. Current: `1`.

New DHT record types can be added without protocol version increment.

---

## 7. Pre-Release Versions

Versions with MAJOR = 0 (e.g., 0.1, 0.2) are pre-release. Pre-release versions:

- Do NOT carry backward compatibility guarantees
- MAY change in breaking ways between MINOR increments
- MUST NOT be used in production deployments
- Are intended for development, testing, and specification refinement

The transition from 0.x to 1.0 is the first stable release. It will be preceded by a minimum 60-day public review period.

---

## 8. Deprecation Process

### Stage 1: Announcement

A version is announced as deprecated in the CHANGELOG and in node advertisements (via an optional "deprecated_versions" field).

### Stage 2: Warning period (6 months minimum)

Client software begins warning users who connect to nodes running deprecated versions. Node software begins logging when handshakes are initiated using deprecated versions.

### Stage 3: End of support

The deprecated version is removed from the default supported_versions list in client and node software. Nodes MAY continue to support it for backward compatibility but are not required to.

### Stage 4: Removal

The deprecated version is removed from all official software. Nodes that continue to run deprecated versions are flagged in the reputation system.

---

## 9. Version Advertisement

Nodes include their supported version range in their DHT advertisement:

```json
{
  "protocol_versions_min": 1,
  "protocol_versions_max": 2
}
```

Clients use this to pre-filter nodes that support the client's desired protocol version before initiating handshakes.

### Cross-Domain Capability Advertisement

For nodes with cross-domain capabilities:

```json
{
  "protocol_versions_min": 1,
  "protocol_versions_max": 1,
  "extensions": {
    "cover_traffic": true,
    "pluggable_transport": true,
    "multiplexing": true,
    "path_quality_reporting": true,
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
}
```

Clients pre-filter nodes by version compatibility and extension requirements before initiating handshakes.

---

## 10. Changelog Entries for Protocol Changes

All protocol changes MUST be documented in CHANGELOG.md with:

- The version number introduced
- Whether the change is MAJOR or MINOR
- The affected message types or behaviors
- Migration guidance for existing implementations

---

## 11. HTTP Version and BGP-X

HTTP protocol version is independent of BGP-X protocol version:

- **BGP-X native services (.bgpx)**: Use HTTP/2 over BGP-X streams
- **Clearnet exit connections**: Exit nodes negotiate HTTP version with destination (HTTP/3 if supported)

HTTP/2 is selected for .bgpx services because BGP-X already provides reliable ordered delivery. HTTP/3's QUIC would add redundant reliability. HTTP version is not negotiated via BGP-X extension flags — it is a transport-layer decision above BGP-X.

---

## 12. Geographic Plausibility and Versioning

Geographic plausibility scoring (extension flag bit 5) is an **OPTIONAL** feature:

- If a node declares a jurisdiction in its advertisement: geo plausibility scoring applies
- If a node does NOT declare a jurisdiction: geo plausibility scoring does NOT apply

Nodes are NOT required to declare jurisdiction. This is an opt-in privacy/convenience tradeoff.

The geographic plausibility algorithm is versioned independently within the reputation system implementation. See `/control-plane/geo_plausibility.md`.

---

## 13. Unified DHT Versioning

BGP-X operates **one unified DHT** spanning all routing domains:

- All nodes participate in the same Kademlia key space
- Domain filtering is a query parameter, not a separate DHT instance
- DHT protocol versioning is independent of wire protocol versioning

DHT record types are versioned independently within each record's JSON structure.
