# BGP-X Bootstrap Node Specification and Operations

**Version**: 0.1.0-draft
**Status**: Pre-implementation specification
**Last Updated**: 2026-04-24

---

## 1. Role of Bootstrap Nodes

Bootstrap nodes are the first contact point for any new participant joining the BGP-X network. They serve **ONE primary function**: helping new participants populate their DHT routing table by responding to `DHT_FIND_NODE` queries.

Bootstrap nodes serve the **discovery role ONLY**. They are the well-known, stable entry points into the unified BGP-X DHT that enable new nodes to find other network participants.

### What Bootstrap Nodes ARE

- **DHT discovery nodes**: Maintain large routing tables and respond to DHT queries
- **Well-known entry points**: Hardcoded in BGP-X software distribution
- **Stable infrastructure**: High-uptime, geographically distributed, publicly reachable
- **DHT record storage nodes**: Store and serve node advertisements, pool advertisements, domain bridge records, and mesh island records

### What Bootstrap Nodes ARE NOT

- **Trusted authorities** (they are not directory authorities like Tor)
- **Relay nodes for user traffic** (they MUST NOT carry onion-encrypted data for paths)
- **Exit nodes** (they do NOT provide clearnet exit functionality)
- **Entry nodes** (they do NOT accept client connections for overlay paths)
- **Single points of failure** (multiple bootstrap nodes exist; network functions without any specific one)

### Independence After Bootstrap

A participant who has successfully joined the DHT no longer depends on bootstrap nodes for any subsequent operation. After the initial routing table population, the node discovers other nodes through normal DHT operations and establishes relay paths using regular relay nodes — not bootstrap nodes.

### Discovery Role Configuration

Bootstrap nodes run a stripped-down version of `bgpx-node` with ONLY the `discovery` role enabled:

```toml
[node]
roles = ["discovery"]
# Bootstrap nodes do NOT serve as relay, entry, or exit
# They participate in DHT discovery only
```

This minimizes the attack surface and resource requirements for bootstrap nodes while ensuring they cannot be used as relays in client path construction.

---

## 2. Bootstrap Node Requirements

### 2.1 Operational Requirements

| Requirement | Minimum Specification |
|---|---|
| **Uptime SLA** | 99.5% or better (99.9% recommended) |
| **Geographic distribution** | Minimum one per major region (NA, EU, AP, SA, AF, ME) |
| **Network connectivity** | Direct BGP peering or colocation at major IXP |
| **Bandwidth** | Minimum 100 Mbps symmetric dedicated |
| **IPv4** | Required |
| **IPv6** | Required |
| **DDoS protection** | Minimum 10 Gbps scrubbing capability |

### 2.2 Hardware Requirements

| Requirement | Minimum | Recommended |
|---|---|---|
| **CPU** | 2 cores | 4+ cores |
| **RAM** | 2 GB | 4 GB |
| **Storage** | 10 GB (for DHT record storage) | 20 GB SSD |
| **Network** | 1 Gbps | 10 Gbps |

### 2.3 DHT Storage Capacity

Bootstrap nodes store DHT records for the unified DHT:

| Record Type | Maximum Stored |
|---|---|
| Node advertisements | 50,000 |
| Pool advertisements | 10,000 |
| Domain bridge records | 5,000 |
| Mesh island records | 2,000 |
| Pool key rotation records | 20,000 |

Eviction on overflow: least recently accessed first.

### 2.4 Security Requirements

- **Key storage**: Bootstrap node private keys MUST be stored in a hardware security module (HSM) or equivalent (Nitrokey HSM, YubiHSM2, cloud HSM)
- **Operator verification**: Bootstrap nodes MUST be operated by the BGP-X project or formally vetted community operators with verified identity
- **Address stability**: Bootstrap node IP addresses MUST be stable (changes require a coordinated software release)
- **Public verification**: Bootstrap nodes MUST publish their NodeID and Ed25519 public key publicly for independent verification
- **No traffic logging**: Bootstrap nodes MUST NOT log client IPs, DHT query contents, or any identifying information
- **Key compromise procedure**: If a bootstrap node's private key is compromised, the node MUST be immediately removed from the hardcoded registry via emergency software release

---

## 3. Software Configuration

### 3.1 Minimal Discovery-Only Configuration

```toml
[node]
roles = ["discovery"]
name = "bgpx-bootstrap-eu-1"
private_key_path = "/etc/bgpx/bootstrap_key"
key_backend = "tpm"  # Or "software" if HSM not available

[[routing_domains]]
domain_type = "clearnet"
domain_id = "0x00000001-00000000"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
public_addr = "203.0.113.1"
public_port = 7474

[network]
listen_addr = "0.0.0.0"
listen_port = 7474
listen_addr_v6 = "::"
public_addr = "203.0.113.1"
public_addr_v6 = "2001:db8::1"
# IPv4 and IPv6 both required

[dht]
max_stored_records = 87000  # Sum of all record type limits
bootstrap_nodes = []  # Bootstrap nodes don't need other bootstraps
k = 20  # Kademlia k parameter
alpha = 3  # Parallel lookup count
advertisement_validity_hours = 24
pool_advertisement_validity_hours = 168  # 7 days for pools
record_expiry_grace_hours = 24

[sessions]
max_sessions = 50000  # Bootstrap nodes handle many concurrent DHT queries

[mesh_island]
auto_publish_advertisement = false  # Bootstrap nodes don't manage islands

[firmware]
variant = "bgpx-gateway"  # Gateway firmware variant recommended
```

### 3.2 Key Management

```bash
# Bootstrap key generation (HSM-backed)
bgpx-keygen --backend tpm --output /etc/bgpx/bootstrap_key

# Or software-backed (less secure, acceptable for testing)
bgpx-keygen --backend software --output /etc/bgpx/bootstrap_key

# Extract NodeID and public key for registry publication
bgpx-keytool inspect /etc/bgpx/bootstrap_key --output public-info.json

# The public-info.json contains:
# {
#   "node_id": "a3f2b9c1d4e5...",
#   "public_key": "base64url-encoded key",
#   "created_at": "2026-04-24T12:00:00Z"
# }
```

---

## 4. Bootstrap Node Registry

The bootstrap node registry is hardcoded in the BGP-X software distribution. It is a list of `(NodeID, IPv4, IPv6, port, region)` tuples.

### 4.1 Hardcoded Registry Format

```toml
# /etc/bgpx/bootstrap_registry.toml (bundled in software distribution)

[[dht.bootstrap_nodes]]
node_id = "a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1"
ipv4 = "203.0.113.1"
ipv6 = "2001:db8::1"
port = 7474
region = "EU"
operator = "bgpx-project"

[[dht.bootstrap_nodes]]
node_id = "b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3"
ipv4 = "198.51.100.1"
ipv6 = "2001:db8:2::1"
port = 7474
region = "NA"
operator = "community-vetted-operator-a"

[[dht.bootstrap_nodes]]
node_id = "c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5"
ipv4 = "203.0.113.2"
ipv6 = "2001:db8:3::1"
port = 7474
region = "AP"
operator = "community-vetted-operator-b"

# Additional bootstrap nodes...
```

### 4.2 Minimum Registry Distribution

The hardcoded registry MUST contain at minimum:

| Region | Minimum Nodes |
|---|---|
| North America | 3 |
| Europe | 3 |
| Asia-Pacific | 2 |
| South America | 1 |
| Africa | 1 |
| Middle East | 1 |

**Total minimum: 11 bootstrap nodes across 6+ distinct operators.**

More nodes are better. The BGP-X network can function with as few as 3 bootstrap nodes, but resilience increases with diversity.

### 4.3 DNS-Based Fallback Registry

If all hardcoded bootstrap nodes are unreachable (e.g., due to IP blocking or network partition), clients fallback to DNS-based bootstrap discovery:

```
_bgpx-bootstrap._udp.bgpx.network  TXT  "nodeid@ipv4:port"
_bgpx-bootstrap._udp.bgpx.network  TXT  "nodeid@[ipv6]:port"
```

**DNS resolution uses DoH (DNS-over-HTTPS)** to prevent ISP interception of bootstrap discovery. Default DoH resolver: Quad9 (`https://dns.quad9.net/dns-query`).

Example DNS query flow:
```
1. Client fails to reach all hardcoded bootstrap IPs
2. Client queries: _bgpx-bootstrap._udp.bgpx.network TXT via DoH
3. DNS returns: ["a3f2b9c1d4e5f6a7b8c9@203.0.113.1:7474", "..."]
4. Client attempts bootstrap using DNS-discovered nodes
5. If successful, DHT routing table populated
6. If DNS also blocked: bootstrap fails, node cannot join network
```

### 4.4 Operator Diversity Requirement

No single operator SHOULD control more than 20% of the bootstrap node registry. If the BGP-X project operates more than 20% of bootstrap nodes, additional community-operated nodes MUST be added to restore diversity.

---

## 5. Bootstrap for Mesh-Only Nodes

Mesh-only nodes (Deployment Mode 4 — no ISP connection) bootstrap via **broadcast beacons**, NOT hardcoded internet bootstrap nodes.

### 5.1 Mesh Bootstrap Procedure

```
1. Broadcast MESH_BEACON on all active mesh transports (WiFi 802.11s, LoRa, BLE)
2. Listen for MESH_BEACON responses from nearby nodes
3. Verify beacon signatures (Ed25519, using beacon's public key)
4. Extract routing table information from verified beacons
5. Build local DHT routing table from verified neighbors
6. Publish own node advertisement to mesh DHT partition
7. Bootstrap complete when routing table has ≥ 5 verified entries
```

### 5.2 No Internet Required for Mesh Bootstrap

Mesh-only nodes do NOT need internet connectivity to bootstrap. The mesh island's DHT partition is reachable entirely via mesh radio transport.

### 5.3 Mesh Island DHT Partition Access

Mesh-only nodes participate in the **unified BGP-X DHT**, but access it differently:

- **Internet-connected nodes**: Access DHT directly via UDP/IP to DHT storage nodes
- **Mesh-only nodes**: Access DHT via domain bridge nodes (mesh gateways) that cache DHT records and serve them over mesh radio transport
- **Bootstrap nodes are NOT mesh gateways**: Bootstrap nodes serve internet-connected nodes only

When a mesh island has a gateway node with internet connectivity, that gateway caches DHT records from the unified DHT and serves queries from mesh-only nodes via radio.

---

## 6. Cross-Domain Bootstrap Prefetch

After initial DHT routing table population, clients with cross-domain path interest (e.g., they regularly connect to services in mesh islands or use specific domain bridges) should prefetch additional records to reduce latency for future cross-domain path construction.

### 6.1 Prefetch Algorithm

```python
def bootstrap_cross_domain(dht_client, config):
    """
    Prefetch domain bridge and mesh island records for common cross-domain paths.
    Called after initial bootstrap completes.
    """
    
    # Prefetch domain bridge records for commonly used domain pairs
    common_bridge_pairs = config.get("cross_domain.common_bridge_pairs", [
        ("clearnet", "mesh:community-island-1"),
        ("clearnet", "mesh:community-island-2"),
    ])
    
    for (from_domain_str, to_domain_str) in common_bridge_pairs:
        from_domain_id = resolve_domain_id(from_domain_str)
        to_domain_id = resolve_domain_id(to_domain_str)
        
        # Domain bridge records use symmetric key ordering
        sorted_pair = sorted([from_domain_id.bytes, to_domain_id.bytes])
        bridge_key = BLAKE3("bgpx-domain-bridge-v1" || sorted_pair[0] || sorted_pair[1])
        
        bridge_record = dht_client.get(bridge_key)
        if bridge_record:
            cache_bridge_record(bridge_record)
        else:
            log_debug(f"No bridge record for {from_domain_str} ↔ {to_domain_str}")

    # Prefetch mesh island advertisements for known islands
    known_islands = config.get("cross_domain.known_mesh_islands", [])
    
    for island_id in known_islands:
        island_key = BLAKE3("bgpx-mesh-island-v1" || island_id.encode("utf-8"))
        island_record = dht_client.get(island_key)
        if island_record:
            cache_island_advertisement(island_record)
```

### 6.2 Prefetch Record TTL

| Record Type | DHT TTL | Re-publication Interval | Client Cache Duration |
|---|---|---|---|
| Domain bridge records | 24 hours | Every 8 hours | 8 hours |
| Mesh island advertisements | 24 hours | Every 8 hours | 8 hours |

Clients SHOULD re-fetch prefetch records when cache entries expire or when cross-domain path construction fails.

---

## 7. Bootstrap Node Operations

### 7.1 Deployment Checklist

- [ ] Node private key generated and backed up to HSM
- [ ] Key backend configured (`key_backend = "tpm"` or `"software"`)
- [ ] Configuration: `roles = ["discovery"]` only — NO relay, entry, or exit roles
- [ ] Firewall: UDP 7474 inbound (IPv4 and IPv6); no relay traffic egress
- [ ] DDoS protection configured (10 Gbps+ scrubbing)
- [ ] Both IPv4 and IPv6 publicly reachable
- [ ] Public addresses configured in `public_addr` and `public_addr_v6`
- [ ] Prometheus monitoring configured and tested
- [ ] Alerting configured (see Section 7.2)
- [ ] On-call rotation established for 30-minute SLA response
- [ ] NodeID and public key published for independent verification
- [ ] DNS TXT record published (if DNS fallback supported)
- [ ] Incident response procedure documented
- [ ] Graceful shutdown tested (publishes NODE_WITHDRAW before drain)

### 7.2 Monitoring and Alerting

Bootstrap nodes MUST implement the following monitoring:

```yaml
# Prometheus alerting rules for bootstrap nodes
groups:
  - name: bgpx_bootstrap
    rules:
      - alert: BootstrapNodeDown
        expr: up{job="bgpx-bootstrap"} == 0
        for: 5m
        severity: critical
        labels:
          team: infrastructure
        annotations:
          summary: "Bootstrap node {{ $labels.instance }} is down"
          description: "Bootstrap node has been unreachable for 5 minutes"

      - alert: BootstrapDHTRoutingTableSmall
        expr: bgpx_dht_routing_table_size < 50
        for: 15m
        severity: warning
        annotations:
          summary: "Bootstrap node DHT routing table is too small"
          description: "Bootstrap nodes should maintain large routing tables (expected > 100)"

      - alert: BootstrapHighQueryRate
        expr: rate(bgpx_dht_queries_received_total[5m]) > 10000
        severity: warning
        annotations:
          summary: "Unusually high DHT query rate"
          description: "May indicate DDoS or surge in network joins"

      - alert: BootstrapAdvertisementExpiringSoon
        expr: bgpx_advertisement_expires_at - time() < 3600
        severity: warning
        annotations:
          summary: "Bootstrap node advertisement expiring within 1 hour"
          description: "Advertisement should be re-published automatically"

      - alert: BootstrapStorageNearLimit
        expr: bgpx_dht_stored_records / bgpx_dht_max_records > 0.90
        severity: warning
        annotations:
          summary: "Bootstrap node DHT storage near capacity"
          description: "Consider increasing max_stored_records or adding more bootstrap nodes"

      - alert: BootstrapKeyHSMDisconnected
        expr: bgpx_hsm_connected == 0
        for: 1m
        severity: critical
        annotations:
          summary: "HSM disconnected from bootstrap node"
          description: "Cannot sign advertisements or withdrawals without HSM"
```

### 7.3 Incident Response

If a bootstrap node becomes unavailable:

1. **Detection**: Automated monitoring alerts on-call operator within 5 minutes
2. **Initial Assessment**: Operator investigates within 15 minutes
3. **Resolution Target**: Issue resolved within 30 minutes (SLA target)
4. **Escalation**: If resolution will take > 30 minutes, escalate to BGP-X infrastructure team
5. **Permanent Retirement**: If bootstrap node must be permanently retired, coordinate software release to update the hardcoded registry

If multiple bootstrap nodes fail simultaneously:
- The remaining bootstrap nodes continue operating
- Network continues to function (not a single point of failure)
- If < 3 bootstrap nodes remain operational, emergency addition of new bootstrap nodes is triggered

### 7.4 Graceful Shutdown Procedure

Bootstrap nodes MUST publish NODE_WITHDRAW before shutdown:

```bash
# Initiate graceful shutdown
bgpx-cli node withdraw --reason maintenance

# Wait for DHT propagation (up to 30 seconds)
# The NODE_WITHDRAW propagates to 5 nearest DHT peers

# Then stop the daemon
systemctl stop bgpx-bootstrap
```

The NODE_WITHDRAW message ensures:
- Storage nodes remove the bootstrap node's advertisement immediately
- Routing nodes remove from DHT k-buckets
- New nodes joining during the shutdown window attempt other bootstrap nodes

---

## 8. Adding and Removing Bootstrap Nodes

### 8.1 Adding a New Bootstrap Node

1. **New operator applies**: Operator generates keypair, publishes NodeID + Ed25519 public key
2. **Identity verification**: BGP-X infrastructure team verifies operator identity and hardware security
3. **Shadow mode**: New node operates in DHT but NOT in software registry for 30 days to establish operational track record
4. **Review**: After 30-day shadow period, infrastructure team reviews:
   - Uptime statistics
   - Query handling performance
   - Security incident history
   - HSM/key management audit
5. **Registry addition**: NodeID added to hardcoded registry in next software release
6. **Deployment**: Software release published with updated registry
7. **Continuity**: Old software releases continue working (existing bootstrap nodes still present in their registries)

### 8.2 Removing a Bootstrap Node

1. **Announcement**: Announce removal at least 90 days in advance (public announcement + direct notification to infrastructure team)
2. **Software update**: Release software update with node removed from hardcoded registry
3. **Adoption period**: Wait for majority of deployed software to update (estimated 60–90 days)
4. **Graceful shutdown**: Bootstrap node publishes NODE_WITHDRAW and drains sessions
5. **Decommission**: Hardware can be repurposed or retired

**CRITICAL**: Bootstrap node removal MUST NOT happen without a prior software release. If a bootstrap node fails unexpectedly and must be immediately taken offline, the remaining nodes continue operating. A software release removing the failed node is published as soon as practical.

### 8.3 Emergency Removal

If a bootstrap node is compromised or must be removed immediately:

1. Immediately remove from DNS TXT records
2. Publish emergency software release with node removed from hardcoded registry
3. Issue security advisory if private key was compromised
4. Monitor for attempts to use the compromised node's NodeID (detect via advertisement signature verification)

---

## 9. Satellite Bootstrap Connectivity

Some bootstrap nodes may use satellite WAN (for remote geographic locations where fiber/cellular is unavailable).

### 9.1 Satellite Bootstrap Configuration

```json
{
  "node_id": "d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d2e1f0a9b8c7",
  "endpoints": [
    { "protocol": "udp", "address": "203.0.113.10", "port": 7474 }
  ],
  "link_quality_profiles": {
    "satellite": {
      "latency_class": 2,  // 2 = satellite LEO (20-60ms)
      "bandwidth_class": 3
    }
  },
  "region": "SA",
  "country": "BR",
  "notes": "Remote Amazon region; Starlink WAN"
}
```

### 9.2 Latency Expectations

| Satellite Type | Expected RTT | Bootstrap Suitability |
|---|---|---|
| Starlink LEO | 20-40ms | Excellent |
| OneWeb LEO | 30-50ms | Excellent |
| Iridium Certus LEO | 150-300ms | Acceptable |
| Inmarsat GEO | 600ms+ | Acceptable for bootstrap (not for relay) |

High latency is acceptable for bootstrap purposes because bootstrap is a one-time operation during node startup. After bootstrap, subsequent DHT queries go to whichever nodes are fastest (not necessarily bootstrap nodes).

### 9.3 Satellite Bootstrap Node Considerations

- **Higher KEEPALIVE timeout**: Configure `keepalive_interval_seconds = 120` for satellite WAN links
- **Geographic plausibility exemption**: Satellite nodes exempt from RTT-based geo plausibility scoring
- **Not used as relays**: Bootstrap nodes are never used as relays regardless of transport
- **Gateway role**: A satellite-connected bootstrap node is still only discovery role; it does NOT act as a gateway for mesh traffic

---

## 10. Multiple Protocol Versions

Bootstrap nodes MUST support the current protocol version AND the immediately preceding MAJOR version.

### 10.1 Version Support Matrix

| Bootstrap Node Version | Supports Client Versions |
|---|---|
| v1.0.x | v1.0.x |
| v1.1.x | v1.1.x, v1.0.x |
| v2.0.x | v2.0.x, v1.x.x |

Version negotiation occurs per client connection via the common header Version field. If a client presents an unsupported version, the bootstrap node responds with ERROR code `PROTOCOL_VERSION_UNSUPPORTED`.

### 10.2 Graceful Version Transitions

When a new MAJOR version is released:
1. Bootstrap nodes are upgraded to support both old and new versions
2. Clients gradually update to new version
3. After 90% adoption of new version, support for old MAJOR version can be deprecated
4. Bootstrap nodes update to support only current MAJOR

---

## 11. Unified DHT Participation

Bootstrap nodes participate in the **unified BGP-X DHT** spanning all routing domains.

### 11.1 DHT Key Space

All BGP-X nodes (bootstrap, relay, mesh, gateway) participate in the same 256-bit Kademlia key space. There is no separate mesh DHT or satellite DHT.

### 11.2 Bootstrap Node DHT Role

Bootstrap nodes are **storage-heavy** DHT nodes:
- Maintain larger-than-average routing tables (k=20 minimum, k=40+ recommended)
- Store more DHT records than typical relay nodes
- Respond to DHT_FIND_NODE queries from any routing domain

### 11.3 Mesh DHT Access via Bootstrap

Mesh-only nodes access the unified DHT through gateway nodes (domain bridge nodes with mesh + clearnet connectivity), not directly through bootstrap nodes. Bootstrap nodes serve internet-connected nodes only.

---

## 12. Operational Metrics

Bootstrap nodes MUST expose the following Prometheus metrics:

| Metric | Type | Description |
|---|---|
| `bgpx_bootstrap_queries_total` | Counter | Total DHT queries received |
| `bgpx_bootstrap_queries_by_type` | Counter | Queries by type (FIND_NODE, GET, PUT) |
| `bgpx_bootstrap_routing_table_size` | Gauge | Current routing table entries |
| `bgpx_bootstrap_stored_records` | Gauge | Number of DHT records stored |
| `bgpx_bootstrap_storage_bytes` | Gauge | Bytes used for DHT storage |
| `bgpx_bootstrap_uptime_seconds` | Counter | Time since daemon start |
| `bgpx_bootstrap_hsm_connected` | Gauge | 1 if HSM connected, 0 otherwise |
| `bgpx_bootstrap_dns_queries_total` | Counter | DNS fallback queries served |
| `bgpx_bootstrap_client_versions` | Histogram | Distribution of client protocol versions |

---

## 13. Security Considerations

### 13.1 Threat Model

| Threat | Mitigation |
|---|---|
| DDoS on bootstrap node | DDoS scrubbing; high bandwidth; rate limiting |
| Malicious bootstrap node | Multiple bootstrap nodes; operator diversity; not trusted authority |
| Key compromise | HSM storage; emergency removal procedure |
| IP blocking by ISP | DNS fallback; multiple IPs; different jurisdictions |
| Traffic analysis of bootstrap queries | Encrypt all queries; cover traffic not applicable to bootstrap |
| Eclipse attack via Sybil bootstrap | Multiple operators; identity verification; 30-day shadow period |

### 13.2 No Trust in Bootstrap Nodes

Bootstrap nodes are NOT trusted authorities. They cannot:
- Control which nodes appear in a client's DHT view (clients verify all signatures)
- Censor DHT records (clients can query any DHT node, not just bootstrap)
- Link client queries to destinations (bootstrap only sees DHT queries, not application traffic)

### 13.3 After Bootstrap, No Dependence

Once a client has populated its DHT routing table from any source (bootstrap, DNS fallback, mesh beacon, or cached peers), it no longer needs bootstrap nodes. The network cannot be permanently disrupted by bootstrap node failures.

---

## 14. Summary

Bootstrap nodes are infrastructure that enable new nodes to join the BGP-X network by providing initial DHT discovery. They are discovery-only nodes with high uptime, geographic distribution, and strong security practices.

Key properties:
- Discovery role ONLY — no relay, entry, or exit functions
- Hardcoded registry with DNS fallback
- Multiple operators across 6+ regions
- HSM-protected keys
- Graceful addition and removal procedures
- Support for satellite WAN in remote regions
- Participation in unified DHT
- 99.5%+ uptime SLA
- Version negotiation for smooth protocol transitions

The BGP-X network can bootstrap without any specific bootstrap node, making it resilient to individual node failures.
