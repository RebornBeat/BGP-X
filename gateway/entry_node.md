# BGP-X Entry Node Specification

**Version**: 0.1.0-draft

---

## 1. Entry Node Role

An entry node is the first relay hop in a BGP-X path. The entry node:
- Knows the client's transport address (IP + port)
- Does NOT know the destination or any subsequent hops
- Does NOT know path length or total hop count
- Receives path_id in HANDSHAKE_DONE for return routing

---

## 2. What the Entry Node Sees

| Information | Visible to Entry Node |
|---|---|
| Client's transport address (IP:port) | ✅ Yes |
| Session key (shared with client) | ✅ Yes (for this one session) |
| Next relay's transport address | ✅ Yes (to forward) |
| Destination | ❌ No |
| Number of total hops | ❌ No |
| Which routing domains the path traverses | ❌ No |
| Whether the path is cross-domain | ❌ No |
| Content of the traffic | ❌ No (only outer onion layer decrypted) |

---

## 3. Entry Node Configuration

Any relay node can accept entry node connections. The `entry` role is not exclusive — nodes can be relay, entry, and discovery simultaneously.

```toml
[node]
roles = ["relay", "entry", "discovery"]
```

Entry nodes accept new HANDSHAKE_INIT connections from clients. They must have at least one publicly reachable endpoint in their routing_domains.

---

## 4. Multi-Domain Entry Points

BGP-X has three equal entry point types:

**Clearnet entry**: client connects to entry node via UDP. Standard operation.

**Mesh island entry**: client (within a mesh island) connects to a mesh entry node via radio transport. The handshake is domain-agnostic — same format as clearnet, delivered via mesh transport.

**Overlay entry**: client already in the BGP-X overlay can extend a path through the overlay domain.

All three entry types use the same HANDSHAKE_INIT → HANDSHAKE_RESP → HANDSHAKE_DONE sequence. The entry node does not know which domain the client originates from.

---

## 5. Rate Limiting

Entry nodes SHOULD apply rate limiting to new handshake attempts:

```toml
[sessions]
max_new_handshakes_per_source_per_minute = 10
max_sessions = 10000
```

Rate limits apply per source IP (clearnet) or per source NodeID (mesh).

---

## 6. Entry Node Privacy

The entry node's knowledge of the client's transport address is unavoidable — the entry node must receive packets from the client. This is the fundamental privacy trade-off of onion routing.

Mitigations:
- Client can use pluggable transport to obfuscate their IP from ISP-level observers
- Client can use a VPN or proxy in front of BGP-X to hide their IP from the entry node
- Pool-based selection and session rotation change entry nodes periodically

---

## 7. Availability Requirements

Entry nodes are the bottleneck for initial connection setup. Recommended uptime: ≥99.0%. BGP-X clients select entry nodes using reputation-weighted random selection — low-uptime nodes are selected less frequently.
