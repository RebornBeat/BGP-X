# BGP-X Pool Operator Guide

**Version**: 0.1.0-draft

---

## 1. What is a Pool?

A BGP-X pool is a named, signed collection of relay nodes grouped by trust level or operational purpose. Pool operators:
- Publish a list of trusted nodes that have been verified
- Sign the pool advertisement with an Ed25519 curator key
- Allow BGP-X users and applications to select paths that only use their trusted nodes

Pools are how trust is organized in BGP-X. The default public pool contains nodes that have met minimum certification standards. Private pools can contain nodes operated by a specific organization or community.

All pool advertisements are stored in the **Unified DHT** — a single distributed hash table that spans all BGP-X routing domains. There is no separate "pool DHT". Pool advertisements are keyed by `BLAKE3("bgpx-pool-v1:" || pool_id)`.

---

## 2. Pool Types

| Type | Description | Access |
|---|---|---|
| **Public** | Open to all users; listed in DHT publicly | Any BGP-X user |
| **Curated** | Operator-vetted nodes; trust above default pool | Any user who adds the pool |
| **Private** | Invitation-only nodes; shared privately | By curator's choice (out-of-band distribution) |
| **Semi-private** | Discoverable in DHT but restricted join | Request-based access |
| **Ephemeral** | Temporary, session-scoped pools | Session participants |
| **Domain-scoped** | Restricted to nodes in a specific routing domain | Any user targeting that domain |
| **Domain-bridge** | Composed of bridge nodes for a specific domain pair | Any user needing that bridge pair |

---

## 3. Creating a Pool

### Step 1: Generate Pool Curator Key

The pool curator key is an Ed25519 keypair separate from your node identity. It is used solely for signing pool advertisements.

```bash
bgpx-keygen --pool-curator --output /etc/bgpx/pool-curator.key

# Show curator public key fingerprint
bgpx-keytool show-public --key /etc/bgpx/pool-curator.key --format fingerprint
# Output: sha256:a3f2b9c1...
```

**Security**: Keep the curator private key secure. If compromised, run emergency key rotation (see Section 7). Consider storing the private key offline or in a hardware security module (HSM) for high-value pools.

### Step 2: Create Pool Advertisement

Pool advertisements are JSON documents with a canonical form for signing.

```json
{
  "pool_id": "privacy-community-exits",
  "pool_type": "curated",
  "version": 1,
  "description": "Curated exit nodes operated by the Privacy Community",
  "curator_pubkey": "a3f2b9c1...",
  "members": [
    "node_id_1_hex",
    "node_id_2_hex",
    "node_id_3_hex"
  ],
  "domain_scope": "clearnet",
  "regions_covered": ["EU", "NA"],
  "created_at": 1737000000,
  "expires_at": 1744000000,
  "policy": {
    "min_uptime_percent": 99.0,
    "require_tier": 3,
    "require_logging_policy_none": true,
    "require_ech": true
  },
  "curator_signature": ""
}
```

**Field Reference**:
- `pool_id`: Unique identifier. Should be descriptive and stable.
- `pool_type`: One of the types listed in Section 2.
- `version`: Increments on every change.
- `members`: Array of NodeIDs (hex-encoded). NodeID = `BLAKE3(Ed25519_public_key)`.
- `domain_scope`: Optional. Restricts membership to a specific routing domain (e.g., `clearnet`, `mesh:lima-district-1`).
- `regions_covered`: Optional. Informational list of regions; does not enforce routing constraints.
- `policy`: Optional. Declares minimum standards for pool membership.

### Step 3: Sign the Advertisement

```bash
bgpx-poolsign \
    --key /etc/bgpx/pool-curator.key \
    --pool pool-advertisement.json \
    --output pool-advertisement-signed.json
```

The signature covers the canonical JSON of all other fields. The `curator_signature` field is populated with the Ed25519 signature.

### Step 4: Publish to DHT

```bash
bgpx-cli pools publish --file pool-advertisement-signed.json

# Verify published
bgpx-cli pools lookup --pool-id privacy-community-exits
```

Pool advertisements are stored in the Unified DHT with a TTL (typically 7 days). Curators should re-publish every 3 days to ensure availability.

---

## 4. Adding and Removing Members

### Adding a Node

Verify the node meets your pool's standards first:

```bash
# Check node reputation
bgpx-cli reputation show <node_id>

# Check node certification tier
bgpx-cli cert show <node_id>

# Verify node is actually running and reachable
bgpx-cli node ping <node_id>

# For domain-bridge pools: verify bridge capability
bgpx-cli domains bridges --node <node_id>
```

**Verification Checklist**:
1. **Reputation**: Sufficient score, no recent protocol violations.
2. **Certification**: Meets minimum tier requirement for your pool.
3. **Availability**: Responds to ping; advertised uptime meets threshold.
4. **Policy Compliance**: Logging policy, ECH support, etc.
5. **Domain/Role**: For domain-scoped pools, confirm node is in the correct domain. For domain-bridge pools, confirm bridge capability.

If the node meets your standards, add it to the pool advertisement and re-sign:

```bash
# Edit pool-advertisement.json: add node_id to members array

# Re-sign with incremented version
bgpx-poolsign \
    --key /etc/bgpx/pool-curator.key \
    --pool pool-advertisement.json \
    --output pool-advertisement-signed.json

# Republish (overwrites previous record)
bgpx-cli pools publish --file pool-advertisement-signed.json
```

### Removing a Node

Edit `members` array to remove the node, increment `version`, re-sign, republish.

Nodes removed from a pool continue operating. BGP-X clients refresh pool membership periodically — after the next refresh (typically within 24 hours), removed nodes are no longer used in paths that require pool membership.

---

## 5. Domain-Scoped Pools

A domain-scoped pool restricts membership to nodes operating in a specific routing domain:

```json
{
  "pool_id": "lima-island-relays",
  "pool_type": "curated",
  "domain_scope": "mesh:lima-district-1",
  "members": ["mesh_node_id_1", "mesh_node_id_2", "mesh_node_id_3"],
  ...
}
```

**Domain Scope Format**: The `domain_scope` field accepts:
- `clearnet`
- `bgpx-overlay`
- `mesh:<island_id>` (e.g., `mesh:lima-district-1`)
- `lora-regional:<region_id>`

When a user requests a path with this pool in the mesh segment, only nodes operating in `mesh:lima-district-1` are considered. Non-mesh nodes cannot join this pool regardless of other qualifications.

**Note**: Domain-scoped pools are stored in the same Unified DHT as all other records. The `domain_scope` is a filter applied during path construction, not a separate DHT partition.

---

## 6. Domain-Bridge Pools

A domain-bridge pool contains bridge nodes for a specific domain pair. These are nodes with endpoints in **two or more routing domains**.

```json
{
  "pool_id": "trusted-lima-bridges",
  "pool_type": "domain_bridge",
  "bridges": {
    "from_domain": "clearnet",
    "to_domain": "mesh:lima-district-1"
  },
  "members": ["bridge_node_id_1", "bridge_node_id_2"],
  ...
}
```

**Verification for Bridge Pool Members**:
Each member must be verified as `bridge_capable` for the specified domain pair:

```bash
# Verify node is a domain bridge
bgpx-cli nodes show <node_id> --json | jq '.bridge_capable'
# Should be true

# Verify specific bridge pair is served
bgpx-cli nodes show <node_id> --json | jq '.bridges'
# Should contain the from_domain -> to_domain pair
```

Users who need cross-domain paths to Lima District 1 mesh can use this pool to ensure their bridge node comes from a trusted set.

---

## 7. Pool Curator Key Rotation

Key rotation is critical for maintaining pool security. Rotate curator keys annually or immediately upon suspected compromise.

### Standard Rotation (Annual or Scheduled)

Standard rotation uses a **dual-signature record** to ensure continuity during the transition.

1. **Generate new key**:
    ```bash
    bgpx-keygen --pool-curator --output /etc/bgpx/pool-curator-new.key
    ```

2. **Create rotation record** (signed by both old and new keys):
    ```bash
    bgpx-poolrotate \
        --old-key /etc/bgpx/pool-curator.key \
        --new-key /etc/bgpx/pool-curator-new.key \
        --pool-id privacy-community-exits \
        --window-hours 24 \
        --output rotation-record.json
    ```

3. **Publish rotation record**:
    ```bash
    bgpx-cli pools publish-rotation --file rotation-record.json
    ```

4. **Wait for acceptance window (24 hours)**: During this time, both keys are valid. Clients update their curator key caches.

5. **Replace old key**:
    ```bash
    mv /etc/bgpx/pool-curator-new.key /etc/bgpx/pool-curator.key
    shred -u /etc/bgpx/pool-curator.key.old 2>/dev/null || true
    ```

6. **Re-sign pool advertisement** with new key and republish:
    ```bash
    bgpx-poolsign \
        --key /etc/bgpx/pool-curator.key \
        --pool pool-advertisement.json \
        --output pool-advertisement-signed.json
    bgpx-cli pools publish --file pool-advertisement-signed.json
    ```

### Emergency Rotation (Key Compromise)

If the curator key is suspected or confirmed compromised, perform an emergency rotation with a shorter acceptance window:

1. **Generate new key immediately**:
    ```bash
    bgpx-keygen --pool-curator --output /etc/bgpx/pool-curator-emergency.key
    ```

2. **Create emergency rotation** (2-hour window):
    ```bash
    bgpx-poolrotate \
        --old-key /etc/bgpx/pool-curator.key \  # if still accessible
        --new-key /etc/bgpx/pool-curator-emergency.key \
        --pool-id privacy-community-exits \
        --window-hours 2 \
        --compromise-confirmed true \
        --output rotation-emergency.json
    ```

3. **Publish and re-sign**:
    ```bash
    bgpx-cli pools publish-rotation --file rotation-emergency.json
    mv /etc/bgpx/pool-curator-emergency.key /etc/bgpx/pool-curator.key
    bgpx-poolsign --key /etc/bgpx/pool-curator.key --pool pool-advertisement.json --output pool-advertisement-signed.json
    bgpx-cli pools publish --file pool-advertisement-signed.json
    ```

4. **Notify pool members and users out-of-band** (email, community channels, etc.).

---

## 8. Using Your Pool

Users add your pool to their BGP-X daemon configuration:

```bash
# Add a pool by ID (fetches advertisement from Unified DHT)
bgpx-cli pools add --pool-id privacy-community-exits

# Or add from file (for private pools distributed out-of-band)
bgpx-cli pools add --file pool-advertisement-signed.json
```

### Configuring Paths to Use Your Pool

Users configure their routing policy or SDK application to use specific pools for path segments.

**Routing Policy (TOML)**:

```toml
# In client config.toml:
[[routing_policy.rules]]
destination = "sensitive-site.com"
action = "bgpx"
path_config = """
  segments:
    - pool: "bgpx-default"
      hops: 3
    - pool: "privacy-community-exits"
      hops: 1
      is_exit: true
  require_ech: true
"""
```

**SDK (Rust)**:

```rust
use bgpx_sdk::{Client, PathConfig, SegmentConfig, SegmentConstraints};

let path = PathConfig {
    segments: vec![
        SegmentConfig {
            pool: "bgpx-default".into(),
            hops: 3,
            constraints: SegmentConstraints::default(),
            ..Default::default()
        },
        SegmentConfig {
            pool: "privacy-community-exits".into(),
            hops: 1,
            is_exit: true,
            constraints: SegmentConstraints {
                require_ech: true,
                ..Default::default()
            },
            ..Default::default()
        },
    ],
    ..Default::default()
};

let stream = client.connect_stream_with_path("bgpx://destination_service_id", path, Default::default()).await?;
```

---

## 9. Pool Governance

For community pools with multiple operators:

- **Multi-signature model**: Decisions to add/remove members require M of N operator signatures (enforced off-chain; the pool advertisement is signed by the single curator key).
- **Published rules**: Publish pool governance rules alongside the pool advertisement (in the `description` field or a linked `.bgpx` document).
- **Regular reviews**: Review member node uptime, reputation scores, and certification status on a schedule (e.g., monthly).
- **Application process**: Establish a clear process for node operators to apply for membership.

For small personal pools, no governance structure is needed — you are the sole curator.

---

## 10. Responsibilities of Pool Operators

By operating a pool, you implicitly vouch for the nodes you include. Pool users trust your curation. Responsibilities:

- **Verify members**: Check reputation scores, uptime, certification tier, and domain capabilities before adding.
- **Monitor members**: Review member health regularly; remove chronically offline or misbehaving nodes.
- **Rotate keys**: Rotate curator key at least annually.
- **Notify users of changes**: Significant membership changes (removing many nodes, key rotation) should be announced in advance via the pool `description` field update or external channels.
- **Geographic plausibility**: Be aware that nodes may declare jurisdiction optionally. Include nodes with plausible location claims if your pool is jurisdiction-specific.
- **No guaranteed trustworthiness**: Your pool certification does not guarantee node operators are honest — it indicates they meet operational standards. Distributed trust remains the fundamental security model.

---

## 11. Security Best Practices

### Curator Key Security

- **Offline storage**: Store the curator private key on an air-gapped machine or hardware token.
- **HSM**: Use a Hardware Security Module for high-value pools.
- **Access control**: Limit who can access the curator key. Use multi-person authorization for production pools.
- **Audit trail**: Log all key usages and membership changes.

### Pool Advertisement Integrity

- Always verify the `curator_signature` before accepting a pool advertisement.
- Users should only add pools from trusted sources.

---

## 12. Hardware Considerations for Pool Members

Pool members run BGP-X nodes on various hardware. As a curator, you may want to ensure hardware diversity or specify requirements:

| Hardware Tier | Use Case | Suitable For Pool Role |
|---|---|---|
| **BGP-X Router v1** | End-user AIO | Relay, Entry, Exit, Bridge |
| **BGP-X Node v1** | Community relay | Relay, Bridge, Gateway |
| **BGP-X Gateway v1** | Provider exit | Exit, High-capacity Bridge |
| **Third-Party OpenWrt** | Existing routers | Relay, Bridge (with adapters) |

Ensure pool members meet performance expectations for the pool's purpose (e.g., high-throughput exit pools require Gateway-class hardware).

---

## 13. Legal and Liability

Operating a pool implies vouching for members. Consult legal counsel regarding:

- **Jurisdictional implications**: Nodes in your pool may traverse various legal jurisdictions.
- **Exit pools**: Pools containing exit nodes may face additional legal scrutiny.
- **Liability**: BGP-X does not make illegal activities legal. Pool operators are not responsible for user traffic content, but are responsible for the trustworthiness of members.

See `legal/liability.md` for detailed framework.

---

## 14. Troubleshooting

### Pool Not Found in DHT

- Verify publication: `bgpx-cli pools lookup --pool-id <pool_id>`
- Check network connectivity to DHT bootstrap nodes.
- Ensure the TTL has not expired (re-publish if necessary).

### Members Not Used in Paths

- Verify node advertisements are present in DHT: `bgpx-cli nodes show <node_id>`
- Check node uptime and reputation scores.
- Ensure node matches `domain_scope` (if applicable).
- For bridge pools, verify `bridge_capable` is true.

### Key Rotation Issues

- Ensure rotation record is published before replacing old key.
- Wait for the full acceptance window for client caches to update.
- Re-sign and re-publish pool advertisement after rotation completes.

---

## 15. Reference

- `protocol/pool_spec.md` — Full protocol specification for pools.
- `protocol/pool_curator_key_rotation.md` — Detailed key rotation specification.
- `hardware/compatible_hardware.md` — Hardware for pool members.
- `legal/liability.md` — Legal framework.
