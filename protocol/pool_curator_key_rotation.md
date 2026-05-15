# BGP-X Pool Curator Key Rotation Specification

**Version**: 0.1.0-draft

This document specifies the procedure for rotating a BGP-X pool curator's signing key — used when a curator rotates their key for security hygiene or in response to a compromise.

---

## 1. Overview

Pool curators hold Ed25519 private keys that sign pool advertisements and member records. These keys may need rotation:

- Periodic security hygiene (scheduled rotation)
- Suspected key exposure
- Confirmed key compromise
- Operator or custody transfer
- HSM replacement

Without a rotation mechanism, a compromised curator key permanently compromises pool integrity. With dual-signature rotation, the new key is cryptographically bound to the old key, providing continuity verification.

### 1.1 Domain-Scoped Pools

A pool may be scoped to a specific routing domain (e.g., a mesh island pool). Key rotation for domain-scoped pools follows the same procedure, with additional considerations:

- Rotation records are published to the **unified DHT** (not a per-domain DHT)
- Mesh-only nodes receive rotation records via their bridge node's DHT cache
- If a mesh island is offline during the grace period, member re-verification is delayed until the island reconnects
- Curators of domain-scoped pools should coordinate rotation timing with expected island connectivity

For domain bridge pools (pools composed of bridge nodes between two domains), key rotation affects cross-domain path construction. Curators should notify bridge node operators in advance of scheduled rotations.

---

## 2. Key Rotation Record Format

A key rotation record is a signed JSON document stored in the DHT.

```json
{
  "version": 1,
  "record_type": "pool_curator_key_rotation",
  "pool_id": "<BLAKE3 pool ID, hex>",
  "old_public_key": "<base64url Ed25519 public key>",
  "new_public_key": "<base64url Ed25519 public key>",
  "rotation_reason": "scheduled",
  "rotation_timestamp": "2026-04-24T12:00:00Z",
  "effective_at": "2026-04-24T12:00:00Z",
  "accept_until": "2026-04-25T12:00:00Z",
  "new_pool_advertisement": {
    "...": "full pool advertisement signed by new key"
  },
  "signature_by_old_key": "<base64url Ed25519 signature>",
  "signature_by_new_key": "<base64url Ed25519 signature>"
}
```

### 2.1 Field Definitions

| Field | Description |
|---|---|
| version | Always 1 |
| record_type | Always "pool_curator_key_rotation" |
| pool_id | Pool being rotated |
| old_public_key | Current (being replaced) curator public key |
| new_public_key | New curator public key |
| rotation_reason | See Section 3 |
| rotation_timestamp | When the rotation was initiated |
| effective_at | When the new key takes effect (typically = rotation_timestamp) |
| accept_until | Clients reject this record after this time |
| new_pool_advertisement | Complete pool advertisement signed by new key |
| signature_by_old_key | Old key signs canonical JSON (excluding both signatures) |
| signature_by_new_key | New key signs canonical JSON (excluding both signatures) |

### 2.2 Canonical JSON for Signing

Both signatures sign the same canonical JSON:

```
canonical = canonical_json(record, exclude=["signature_by_old_key", "signature_by_new_key"])
```

Canonical JSON: UTF-8, compact, keys sorted alphabetically, both signature fields excluded.

---

## 3. Rotation Reason Codes

| Code | Description | accept_until Window |
|---|---|---|
| `scheduled` | Routine periodic rotation | 24 hours |
| `compromise_suspected` | Possible key exposure | 12 hours |
| `compromise_confirmed` | Confirmed key compromise | 2 hours |
| `operator_transfer` | Pool changing operators | 24 hours |
| `hsm_replacement` | Hardware security module replaced | 24 hours |

The `accept_until` field determines how long clients will accept this rotation record. Emergency rotations use shorter windows to force rapid adoption.

---

## 4. Verification Procedure

```python
function verify_key_rotation(record):

    # Step 1: Format validation
    if record.version != 1 or record.record_type != "pool_curator_key_rotation":
        return REJECT("Invalid format")

    # Step 2: Timing validation
    now = current_utc_time()
    effective_at = parse_iso8601(record.effective_at)
    accept_until = parse_iso8601(record.accept_until)

    if now > accept_until:
        return REJECT("Rotation record expired")

    if now < effective_at - 60_seconds:  # 60s clock skew tolerance
        return REJECT("Rotation record from future")

    if accept_until - effective_at > 25 * 3600:  # Max 25hr window
        return REJECT("Acceptance window exceeds maximum")

    # Step 3: Verify old key matches cached curator key
    cached_curator_key = curator_key_cache.get(record.pool_id)
    if cached_curator_key is None:
        # Never seen this pool — log and proceed with caution
        log("First-time pool learning from rotation record for", record.pool_id)
    elif cached_curator_key != record.old_public_key:
        return REJECT("Old key does not match cached curator key")

    # Step 4: Compute canonical JSON for signature verification
    canonical = canonical_json(record, exclude=["signature_by_old_key", "signature_by_new_key"])
    canonical_bytes = canonical.encode("utf-8")

    # Step 5: Verify old key signature
    if not ed25519_verify(record.old_public_key, record.signature_by_old_key, canonical_bytes):
        return REJECT("Old key signature invalid")

    # Step 6: Verify new key signature (dual-signature requirement)
    if not ed25519_verify(record.new_public_key, record.signature_by_new_key, canonical_bytes):
        return REJECT("New key signature invalid")

    # Step 7: Verify embedded new pool advertisement
    if not verify_pool_advertisement(record.new_pool_advertisement, record.new_public_key):
        return REJECT("New pool advertisement invalid")

    # Step 8: Apply rotation
    curator_key_cache.update(record.pool_id, record.new_public_key)
    pool_advertisement_cache.update(record.pool_id, record.new_pool_advertisement)

    log("Pool curator key rotation accepted for pool:", record.pool_id)
    return ACCEPT
```

---

## 5. Pool Member Re-Verification After Rotation

After a key rotation is accepted:

1. All pool member records signed under the old curator key are flagged as "needs-reverification"
2. **24-hour grace period**: Old-key-signed members remain eligible (with warning logged)
3. After 24 hours: old-key-signed members excluded from pool selection until re-signed
4. The curator should re-sign all member records within 24 hours as part of the rotation procedure

### 5.1 Curator Re-Signing Workflow

```bash
# Step 1: Apply new key
bgpx-cli pool rotate-key \
    --pool-id my-pool \
    --old-key /etc/bgpx/curator_key_old \
    --new-key /etc/bgpx/curator_key_new \
    --reason scheduled

# Step 2: Re-sign all pool members (curator responsibility)
bgpx-cli pool resign-members \
    --pool-id my-pool \
    --key /etc/bgpx/curator_key_new

# Step 3: Re-publish updated pool advertisement
bgpx-cli pool publish --pool-id my-pool

# Step 4: Verify rotation completed
bgpx-cli pool verify --pool-id my-pool
```

The `resign-members` command:
1. Fetches current member list from pool advertisement
2. For each member: generates new member signature using new curator key
3. Publishes updated pool advertisement with all member signatures refreshed

### 5.2 Client Key Rotation Cache (Implementation Reference)

```rust
pub struct CuratorKeyCache {
    // Current active key per pool_id
    active_keys: HashMap<PoolId, Ed25519PublicKey>,

    // Historical keys (for verifying older member signatures within grace period)
    historical_keys: HashMap<PoolId, Vec<(Ed25519PublicKey, SystemTime)>>,

    // Pending rotations (received, effective_at in future)
    pending_rotations: HashMap<PoolId, KeyRotationRecord>,
}

impl CuratorKeyCache {

    pub fn update(&mut self, pool_id: PoolId, new_key: Ed25519PublicKey) {
        // Move old key to historical
        if let Some(old_key) = self.active_keys.remove(&pool_id) {
            self.historical_keys.entry(pool_id.clone())
                .or_default()
                .push((old_key, SystemTime::now()));
        }
        self.active_keys.insert(pool_id, new_key);
    }

    pub fn is_valid_key(&self, pool_id: &PoolId, key: &Ed25519PublicKey) -> KeyValidity {
        if self.active_keys.get(pool_id) == Some(key) {
            return KeyValidity::Current;
        }
        if let Some(historical) = self.historical_keys.get(pool_id) {
            for (hist_key, rotated_at) in historical {
                if hist_key == key {
                    let grace_expired = SystemTime::now()
                        .duration_since(*rotated_at)
                        .map(|d| d.as_secs() > 24 * 3600)
                        .unwrap_or(true);
                    if grace_expired {
                        return KeyValidity::Expired;
                    } else {
                        return KeyValidity::GracePeriod;
                    }
                }
            }
        }
        KeyValidity::Unknown
    }
}

pub enum KeyValidity {
    Current,       // Active key — accept member signatures
    GracePeriod,   // Old key within 24hr grace — accept with warning
    Expired,       // Old key after grace — reject member signatures
    Unknown,       // Not seen before — reject
}
```

### 5.3 Mesh Island Grace Period Handling

If a mesh island is disconnected from the clearnet during a pool's grace period:

1. Bridge nodes cache the rotation record
2. When the island reconnects, bridge nodes propagate the rotation to mesh nodes
3. Grace period countdown pauses while island is offline (honest implementation)
4. Alternatively: curator may extend grace period when scheduling rotation during expected island downtime

Curators should avoid scheduling rotations during expected island maintenance windows.

---

## 6. Emergency Rotation Procedure (Compromise Confirmed)

```bash
# 1. Generate new keypair immediately
bgpx-cli keygen --output /etc/bgpx/curator_key_new
# SECURE the new key immediately (HSM or equivalent)

# 2. Publish emergency rotation (2-hour acceptance window)
bgpx-cli pool rotate-key \
    --pool-id my-pool \
    --old-key /etc/bgpx/curator_key_compromised \
    --new-key /etc/bgpx/curator_key_new \
    --reason compromise_confirmed \
    --accept-window 2h

# 3. Re-sign all members IMMEDIATELY (before 2hr window expires)
bgpx-cli pool resign-members \
    --pool-id my-pool \
    --key /etc/bgpx/curator_key_new

# 4. Publish updated pool advertisement
bgpx-cli pool publish --pool-id my-pool

# 5. IMMEDIATELY destroy the compromised key material
bgpx-cli key destroy /etc/bgpx/curator_key_compromised
# Verify destruction with hardware HSM audit log if available

# 6. Notify BGP-X project of the compromise
# (helps track and respond to systematic attacks)
```

---

## 7. DHT Storage

Key rotation records are stored in the **unified DHT** at:

```
rotation_key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || rotation_timestamp_BE8)
```

The rotation timestamp (uint64 Unix seconds, big-endian 8 bytes) is included in the key to allow multiple rotation records to coexist for audit history.

**Record TTL**: accept_until + 7 days (retained for audit purposes after expiry)

**Retrieval**: clients query DHT_GET for the most recent valid rotation record before trusting a pool with an unknown curator key.

**Mesh Node Access**: Mesh-only nodes receive rotation records via their bridge node's DHT cache when the island is connected to clearnet. If the island is offline during rotation, records propagate on reconnection.

**Domain-Scoped Pools**: Rotation records for domain-scoped pools are still stored in the unified DHT. The pool's `domain_scope` field in the pool advertisement determines which nodes consider this pool during path construction, but all nodes participate in DHT storage.

---

## 8. Client Key Rotation Cache Behavior

Clients maintain:

- Active key per pool_id
- Historical keys with rotation timestamps (for 24-hour grace period)
- Pending rotations (effective_at in future)

Key validity states:

- **Current**: active key — accept member signatures
- **GracePeriod**: old key within 24 hours of rotation — accept with warning
- **Expired**: old key after 24-hour grace — reject member signatures
- **Unknown**: not seen before — reject

---

## 9. Wire Format

```
BGP-X Common Header (32 bytes, Msg Type = 0x1C, Session ID = 0x00...00)
Record length (4 bytes, uint32 BE)
Canonical JSON of rotation record (UTF-8, max 4096 bytes)
```

---

## 10. Cross-Domain Path Construction Impact

When a domain bridge pool's curator key rotates:

1. All bridge nodes in the pool must have their member records re-signed
2. Cross-domain paths using this pool remain valid during the grace period
3. After grace period: bridge nodes not re-signed are excluded from pool selection
4. Path construction will select from re-signed bridge nodes only

**Impact on active cross-domain sessions**: Sessions established before rotation continue normally. The session key is independent of pool curator keys. Only future path construction is affected.

**Operator notification**: Curators rotating domain bridge pool keys SHOULD notify bridge node operators 48 hours in advance for scheduled rotations, allowing operators to prepare for re-signing.

---

## 11. Test Vectors

Required test vectors:

1. **Standard rotation verification**: 3 test cases with valid dual-signatures
2. **Emergency rotation**: 2 test cases (compromise_confirmed, 2hr window)
3. **Expired rotation record**: 2 test cases (accept_until passed)
4. **Key mismatch**: 2 test cases (old_public_key doesn't match cached)
5. **Invalid old signature**: 2 test cases (old key signature fails)
6. **Invalid new signature**: 2 test cases (new key signature fails)
7. **Grace period**: 3 test cases (member signed by old key within / outside grace period)
