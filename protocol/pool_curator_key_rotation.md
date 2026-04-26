# BGP-X Pool Curator Key Rotation Specification

**Version**: 0.1.0-draft

---

## 1. Overview

Pool curators hold Ed25519 private keys that sign pool advertisements and member records. These keys may need rotation for:

- Periodic security hygiene (scheduled rotation)
- Suspected or confirmed key exposure
- Operator or custody transfer
- HSM replacement

Dual-signature rotation records provide cryptographic continuity: the new key is bound to the old key, preventing an adversary with only the new key from backdating a rotation.

---

## 2. Key Rotation Record Format

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

Both signatures sign the canonical JSON excluding both signature fields.

---

## 3. Rotation Reason Codes and Acceptance Windows

| Code | Description | accept_until Window |
|---|---|---|
| `scheduled` | Routine periodic rotation | 24 hours |
| `compromise_suspected` | Possible key exposure | 12 hours |
| `compromise_confirmed` | Confirmed key compromise | 2 hours |
| `operator_transfer` | Pool changing operators | 24 hours |
| `hsm_replacement` | Hardware security module replaced | 24 hours |

---

## 4. Verification Procedure

```python
function verify_key_rotation(record):

    # Format validation
    if record.version != 1 or record.record_type != "pool_curator_key_rotation":
        return REJECT("Invalid format")

    # Timing validation
    now = current_utc_time()
    if now > parse_iso8601(record.accept_until):
        return REJECT("Rotation record expired")
    if now < parse_iso8601(record.effective_at) - 60:
        return REJECT("Rotation record from future")
    if (parse_iso8601(record.accept_until) - parse_iso8601(record.effective_at)) > 25 * 3600:
        return REJECT("Acceptance window exceeds 25 hours maximum")

    # Verify old key matches cached curator key
    cached = curator_key_cache.get(record.pool_id)
    if cached is not None and cached != record.old_public_key:
        return REJECT("Old key does not match cached curator key")

    # Compute canonical JSON for signature verification
    canonical = canonical_json(record, exclude=["signature_by_old_key", "signature_by_new_key"])
    canonical_bytes = canonical.encode("utf-8")

    # Verify both signatures (MANDATORY — dual-signature required)
    if not ed25519_verify(record.old_public_key, record.signature_by_old_key, canonical_bytes):
        return REJECT("Old key signature invalid")

    if not ed25519_verify(record.new_public_key, record.signature_by_new_key, canonical_bytes):
        return REJECT("New key signature invalid")

    # Verify embedded new pool advertisement
    if not verify_pool_advertisement(record.new_pool_advertisement, record.new_public_key):
        return REJECT("New pool advertisement invalid")

    # Apply rotation
    curator_key_cache.update(record.pool_id, record.new_public_key)
    pool_advertisement_cache.update(record.pool_id, record.new_pool_advertisement)

    return ACCEPT
```

---

## 5. Pool Member Re-Verification After Rotation

After rotation is accepted:
1. All pool member records signed under old curator key are flagged "needs-reverification"
2. **24-hour grace period**: old-key-signed members remain eligible (with warning logged)
3. After 24 hours: old-key-signed members excluded until re-signed by curator under new key
4. Curator MUST re-sign all members within 24 hours as part of rotation

---

## 6. Emergency Rotation Procedure (Compromise Confirmed)

```bash
# Generate new keypair immediately
bgpx-cli keygen --output /etc/bgpx/curator_key_new
# SECURE the new key immediately

# Publish emergency rotation (2-hour acceptance window)
bgpx-cli pool rotate-key \
    --pool-id my-pool \
    --old-key /etc/bgpx/curator_key_compromised \
    --new-key /etc/bgpx/curator_key_new \
    --reason compromise_confirmed \
    --accept-window 2h

# Re-sign all members IMMEDIATELY (before 2hr window expires)
bgpx-cli pool resign-members \
    --pool-id my-pool \
    --key /etc/bgpx/curator_key_new

# Re-publish updated pool advertisement
bgpx-cli pool publish --pool-id my-pool

# IMMEDIATELY destroy the compromised key material
bgpx-cli key destroy /etc/bgpx/curator_key_compromised
```

---

## 7. DHT Storage

Key rotation records stored at:
```
key = BLAKE3("bgpx-pool-key-rotation-v1" || pool_id || rotation_timestamp_BE8)
TTL: accept_until + 7 days (retained for audit after expiry)
```

---

## 8. Client Key Rotation Cache

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

## 10. Test Vectors

1. Standard rotation verification (valid dual-signatures): 3 vectors
2. Emergency rotation (compromise_confirmed, 2hr window): 2 vectors
3. Expired rotation record: 2 vectors
4. Old key does not match cached: 2 vectors
5. Invalid old signature: 2 vectors
6. Invalid new signature: 2 vectors
7. Grace period membership: 3 vectors (within/outside 24hr grace)
