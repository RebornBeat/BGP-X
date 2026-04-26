# BGP-X Bootstrap Node Specification

**Version**: 0.1.0-draft

---

## 1. Role of Bootstrap Nodes

Bootstrap nodes serve the discovery role ONLY. They do NOT relay BGP-X traffic, do NOT handle onion packets, do NOT act as exit nodes.

Responsibilities:
- Maintain up-to-date DHT routing tables
- Respond to DHT_FIND_NODE queries
- Store and serve node advertisements, pool advertisements, domain bridge records, mesh island records
- Serve as stable, well-known entry points for new nodes joining the network

---

## 2. Bootstrap Node Requirements

| Requirement | Minimum |
|---|---|
| Uptime | 99.5% |
| Bandwidth | 100 Mbps symmetric |
| RAM | 2 GB |
| CPU | 2 cores |
| Storage | 10 GB (for DHT record storage) |
| IPv4 | Required |
| IPv6 | Required |
| Geographic distribution | ≥6 regions: NA, EU, AP, SA, AF, ME |

---

## 3. Configuration

```toml
[node]
roles = ["discovery"]
name = "bgpx-bootstrap-eu-1"

[[routing_domains]]
domain_type = "clearnet"
enabled = true
listen_addr = "0.0.0.0"
listen_port = 7474
public_addr = "203.0.113.1"
public_port = 7474

[dht]
max_stored_records = 100000
bootstrap_nodes = []  # Bootstrap nodes don't need other bootstraps
k = 20
alpha = 3
advertisement_validity_hours = 24

[mesh_island]
auto_publish_advertisement = false  # Bootstrap nodes don't manage islands
```

---

## 4. Hardcoded Bootstrap List (Client Configuration)

Default BGP-X client configuration includes hardcoded bootstrap node list:

```toml
[[dht.bootstrap_nodes]]
address = "203.0.113.1"
port = 7474
node_id = "a3f2b1c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2"
region = "EU"

[[dht.bootstrap_nodes]]
address = "198.51.100.1"
port = 7474
node_id = "b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3"
region = "NA"
```

DNS fallback: `_bgpx-bootstrap._udp.bgpx.network` TXT records, queried via DoH (Quad9).

---

## 5. Cross-Domain Bootstrap Prefetch

After initial DHT routing table populated, clients with cross-domain interest additionally prefetch:

```python
def bootstrap_cross_domain():
    # Prefetch domain bridge records for commonly used pairs
    common_bridge_pairs = load_from_config_or_defaults()
    for (from_domain, to_domain) in common_bridge_pairs:
        sorted_pair = sorted([from_domain.id_bytes, to_domain.id_bytes])
        bridge_key = BLAKE3("bgpx-domain-bridge-v1" || sorted_pair[0] || sorted_pair[1])
        bridge_record = dht_get(bridge_key)
        if bridge_record:
            cache_bridge_record(bridge_record)

    # Prefetch mesh island advertisements for known islands
    known_islands = load_from_config_or_defaults()
    for island_id in known_islands:
        island_key = BLAKE3("bgpx-mesh-island-v1" || island_id.encode("utf-8"))
        island_record = dht_get(island_key)
        if island_record:
            cache_island_advertisement(island_record)
```

Bridge node records: TTL 24 hours, re-publication every 8 hours.
Island advertisement records: TTL 24 hours, re-publication every 8 hours.

---

## 6. Multiple Bootstrap Protocol Versions

Bootstrap nodes MUST support the current protocol version AND the immediately preceding MAJOR version.

Version negotiation per client connection — unchanged.

---

## 7. Bootstrap Node DHT Record Storage Limits

```
Maximum node advertisements: 50,000
Maximum pool advertisements: 10,000
Maximum domain bridge records: 5,000
Maximum mesh island records: 2,000
Maximum pool key rotation records: 20,000
```

Eviction on overflow: least recently accessed first.

---

## 8. Bootstrap Node Monitoring

Bootstrap nodes publish their own uptime and availability metrics to an operator-controlled dashboard. Bootstrap node health is monitored independently — a bootstrap node that fails DHT queries is removed from the hardcoded list in the next release.
