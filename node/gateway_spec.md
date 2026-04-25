# BGP-X Gateway Specification

**Version**: 0.1.0-draft

This document specifies the BGP-X gateway — the component that bridges the BGP-X overlay network to the public internet. Gateways are the only BGP-X nodes that have contact with both the overlay and the clearnet simultaneously.

---

## 1. Role and Responsibility

A BGP-X gateway (also called an exit node) serves as the translation layer between:

```
BGP-X Overlay (private, encrypted, identity-based)
                        ↕
        Gateway (translation + exit policy enforcement)
                        ↕
  Public Internet (IP-addressed, unencrypted at network level)
```

The gateway is the most security-sensitive component of the BGP-X network because:

- It is the only node that sees destination addresses
- It is the point at which BGP-X traffic enters the public internet
- Its behavior is the most legally consequential for its operator

All gateway behavior must be transparent, policy-driven, and auditable.

---

## 2. Gateway Architecture

```
Overlay side                                    Clearnet side
─────────────────────────────────────────────────────────────
BGP-X UDP listener
        │
        ▼
Onion decryption
(final layer)
        │
        ▼
Exit policy engine ──── DENY ────► STREAM_CLOSE (ERR_EXIT_POLICY_DENIED)
        │
       ALLOW
        │
        ▼
Stream demultiplexer
        │
        ├── TCP streams ──────────► TCP connection manager ──► Internet (TCP)
        │
        └── UDP streams ──────────► UDP socket manager ───────► Internet (UDP)
                                            │
                                            ▼
                                   Response handler
                                            │
                                            ▼
                                   Re-encrypt for return path
                                            │
                                            ▼
                                   BGP-X overlay (return)
```

---

## 3. Gateway Identity

A gateway is a BGP-X node with the "exit" role. It has:

- A node identity (Ed25519 keypair, NodeID)
- An operator identity (operator_id, operator Ed25519 keypair)
- A signed exit policy (signed by operator private key)

The exit policy is the public commitment the gateway makes about its behavior. Clients read the exit policy from the DHT before selecting the gateway for a path.

---

## 4. Exit Policy Enforcement

Exit policy enforcement is the gateway's primary responsibility on the clearnet side.

### 4.1 Enforcement algorithm

```
function evaluate_stream_open(stream_open_msg):

    protocol = determine_protocol(stream_open_msg.flags)
    destination = parse_destination(stream_open_msg.destination)
    port = stream_open_msg.port

    # Protocol check
    if protocol not in exit_policy.allow_protocols:
        return DENY(reason=ERR_EXIT_POLICY_DENIED, detail="protocol")

    # Port check
    if exit_policy.allow_ports != [0]:  # [0] means all ports
        if port not in exit_policy.allow_ports:
            return DENY(reason=ERR_EXIT_POLICY_DENIED, detail="port")

    # Destination check
    resolved_ip = resolve_if_domain(destination)
    if resolved_ip is None:
        return DENY(reason=ERR_DESTINATION_UNREACHABLE, detail="dns_failure")

    for cidr in exit_policy.deny_destinations:
        if resolved_ip in cidr:
            return DENY(reason=ERR_EXIT_POLICY_DENIED, detail="destination")

    # Private IP check (belt and suspenders — also in deny_destinations)
    if is_private_ip(resolved_ip):
        return DENY(reason=ERR_EXIT_POLICY_DENIED, detail="private_destination")

    return ALLOW(resolved_ip=resolved_ip)
```

### 4.2 Policy enforcement logging

When a stream is denied:

- Log: timestamp, denial reason category (protocol/port/destination class), NOT the full destination address
- Increment denial counter by category
- Do NOT log any information identifying which BGP-X session or path was involved

When a stream is allowed:

- Log nothing (no access log by default with `logging_policy = "none"`)

### 4.3 DNS resolution for exit policy

When the destination is a domain name, the gateway resolves it before checking deny_destinations.

DNS resolution uses the gateway's configured resolver (default: system resolver). The gateway SHOULD use an encrypted DNS resolver (DoH or DoT) to prevent DNS-level surveillance.

Important: the resolved IP is used for exit policy enforcement. The domain name itself is also checked against a separate domain blacklist (if configured). If the resolved IP changes between the policy check and the connection establishment, the new IP is re-checked.

---

## 5. Connection Management

### 5.1 TCP connection pool

The gateway maintains a pool of outbound TCP connections per destination:

```
connection_pool: HashMap<DestinationKey, ConnectionPool>

DestinationKey = (resolved_ip: IpAddr, port: u16)

ConnectionPool {
    max_connections: usize,
    idle_connections: VecDeque<TcpStream>,
    active_connections: usize,
    connection_timeout: Duration,
}
```

Connection pooling is transparent to BGP-X clients. From the client's perspective, each stream maps to one logical connection. The gateway may reuse TCP connections to the same destination across multiple BGP-X streams (HTTP/1.1 keep-alive, HTTP/2 multiplexing).

### 5.2 Connection limits

| Limit | Default | Description |
|---|---|---|
| max_outbound_connections | 2,000 | Total outbound connections |
| max_connections_per_destination | 50 | Per destination IP |
| connection_timeout_seconds | 10 | TCP connect timeout |
| idle_connection_timeout_seconds | 60 | Close idle connections after |

### 5.3 UDP handling

UDP streams at the gateway use a dedicated UDP socket per BGP-X stream. The gateway:

1. Opens a local UDP socket (OS assigns ephemeral port)
2. Sends the datagram to the destination
3. Listens on the local socket for responses (with configurable timeout)
4. Forwards responses back through the BGP-X overlay

UDP "connection" timeout: 5 minutes of inactivity (adjustable per exit policy).

---

## 6. Response Handling

Responses from clearnet destinations travel back through the BGP-X path to the client.

### 6.1 Response encryption

The gateway does NOT construct full onion packets for the return path. Instead, it uses the established return session to the previous relay in the path:

```
response_data = read from clearnet connection
encrypted_response = session_key.encrypt(
    sequence = next_outbound_sequence,
    aad = return_header,
    plaintext = response_data
)
udp_send(previous_relay_addr, encrypted_response)
```

Each relay on the return path performs the same operation — encrypting the data for its predecessor and forwarding it. By the time the data reaches the entry node, it has been re-encrypted by each hop.

### 6.2 Response buffering

For TCP responses, the gateway buffers response data before forwarding to handle differences between TCP streaming and BGP-X packet boundaries:

- Buffer size: 64 KB per stream (default)
- Flush trigger: buffer full, FIN received, or 10ms timeout (whichever comes first)
- Maximum response packet size: BGP-X MTU (1160 bytes of application data)

Large responses are segmented into multiple BGP-X packets automatically.

---

## 7. Logging Policy Enforcement

The gateway's logging policy is a core commitment to its users. It MUST be enforced by technical controls, not just policy documents.

### 7.1 "none" logging policy

When `logging_policy = "none"`:

- No access logs are written (no connection records)
- No DNS query logs are written
- Stream denial events log only: timestamp, denial category (no destination)
- Aggregate statistics are maintained in-memory (reset on restart)
- No data is written to disk about individual connections

Technical enforcement:
- No file handles are opened for access logging
- Aggregate stats are stored in `tmpfs` (memory-only filesystem)
- Crash logs (stderr) capture only daemon errors, not traffic events

### 7.2 "metadata" logging policy

When `logging_policy = "metadata"`:

Records per connection:
- Timestamp
- Protocol (TCP/UDP)
- Destination port
- Destination IP (hash, not plaintext) — privacy-preserving
- Bytes transferred
- Duration

Does NOT record:
- Full destination IP or domain
- Content
- BGP-X session or path information

### 7.3 "full" logging policy

When `logging_policy = "full"`:

- Full access log including destination IP, port, bytes, timing
- Intended only for operators with specific legal requirements
- Clients SHOULD avoid gateways with `logging_policy = "full"` for sensitive traffic

---

## 8. Gateway Registration and Verification

### 8.1 Required fields for DHT registration

Before a gateway can be selected by clients, its advertisement must include:

- `"exit"` in roles
- A complete, signed exit policy
- `operator_id` (strongly recommended; some clients may require it)
- `operator_contact` in exit policy
- `jurisdiction` in exit policy

### 8.2 Exit policy signing

The exit policy is signed by the operator's private key (not the node's private key). This separation allows:

- The exit policy to be updated without changing the node identity
- The operator identity to be separate from the node identity
- Multiple nodes operated by the same entity to share an exit policy signature

Signing procedure:

```bash
bgpx-cli exit-policy sign \
    --policy /etc/bgpx/exit_policy.json \
    --operator-key /etc/bgpx/operator_private_key \
    --output /etc/bgpx/exit_policy_signed.json
```

The signed policy is included in the node advertisement automatically.

### 8.3 Client verification of gateway

Before selecting a gateway, a client MUST:

1. Retrieve the gateway's advertisement from DHT
2. Verify the node advertisement signature
3. Verify the exit policy signature (signed by operator_id key)
4. Evaluate the exit policy against the client's requirements
5. Check the gateway's reputation score
6. Verify the gateway is not blacklisted

Only if all checks pass is the gateway eligible for path selection.
