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

---

## 3. Version Negotiation

Version negotiation occurs during the handshake.

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

## 4. Compatibility Rules

### 4.1 MINOR version changes

MINOR version increments are backward-compatible. A node running version 1.3 can communicate with a node running version 1.1:

- New message types added in 1.3 are unknown to 1.1 nodes but MUST be gracefully ignored by 1.1 implementations
- New fields added to existing message types MUST be appended (not inserted) and MUST have safe defaults when absent
- The behavior of all existing message types is preserved

Implementations MUST accept messages with unknown trailing bytes in payloads and MUST NOT reject them.

### 4.2 MAJOR version changes

MAJOR version increments are explicitly breaking. Implementations that support MAJOR version N are not required to support MAJOR version N-1. However:

- Implementations SHOULD support at least one previous MAJOR version for a transition period of no less than 12 months
- A deprecation notice MUST be published no less than 6 months before a MAJOR version becomes the minimum required version
- Bootstrap nodes distributed with client software MUST support the current MAJOR version and the immediately preceding MAJOR version

### 4.3 Extension negotiation

Extensions (cover traffic, pluggable transport, etc.) are negotiated via the extension flags in HANDSHAKE_INIT and HANDSHAKE_RESP, independently of the protocol version. This allows extensions to be deployed without version increments.

---

## 5. Deprecation Process

### Stage 1: Announcement

A version is announced as deprecated in the CHANGELOG and in node advertisements (via an optional "deprecated_versions" field).

### Stage 2: Warning period (6 months minimum)

Client software begins warning users who connect to nodes running deprecated versions. Node software begins logging when handshakes are initiated using deprecated versions.

### Stage 3: End of support

The deprecated version is removed from the default supported_versions list in client and node software. Nodes MAY continue to support it for backward compatibility but are not required to.

### Stage 4: Removal

The deprecated version is removed from all official software. Nodes that continue to run deprecated versions are flagged in the reputation system.

---

## 6. Pre-Release Versions

Versions with MAJOR = 0 (e.g., 0.1, 0.2) are pre-release. Pre-release versions:

- Do NOT carry backward compatibility guarantees
- MAY change in breaking ways between MINOR increments
- MUST NOT be used in production deployments
- Are intended for development, testing, and specification refinement

The transition from 0.x to 1.0 is the first stable release. It will be preceded by a minimum 60-day public review period.

---

## 7. Version Advertisement

Nodes include their supported version range in their DHT advertisement:

```json
{
  "protocol_versions_min": 1,
  "protocol_versions_max": 2
}
```

Clients use this to pre-filter nodes that support the client's desired protocol version before initiating handshakes.

---

## 8. Changelog Entries for Protocol Changes

All protocol changes MUST be documented in CHANGELOG.md with:

- The version number introduced
- Whether the change is MAJOR or MINOR
- The affected message types or behaviors
- Migration guidance for existing implementations
