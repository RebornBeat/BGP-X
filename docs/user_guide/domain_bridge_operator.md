# Domain Bridge Operator

**Version**: 0.1.0-draft

---

## 1. What is a Domain Bridge Operator?

A domain bridge operator runs a BGP-X node with connections to two or more routing domains — typically clearnet (internet) and a mesh island (LoRa or WiFi mesh). The bridge node enables traffic to cross between these domains.

When you operate a domain bridge:
- Clearnet BGP-X users can reach services hosted in your mesh island
- Mesh island members can reach clearnet via your bridge (if you also run an exit)
- Your bridge node appears in the unified DHT so path construction algorithms can discover it

**Privacy of your bridge node**: your bridge node sees only the IP address of the last clearnet relay that forwarded traffic to you, and the LoRa node ID of the first mesh relay in the island. It cannot see the original client's IP or the service's identity. Inner onion layers remain encrypted.

---

## 2. Hardware Requirements for Bridge Operation

A domain bridge requires two simultaneous network connections: one to clearnet (any internet connection) and one to a mesh domain (radio).

### 2.1 Primary Hardware Options

**Option A: Purpose-Built BGP-X Hardware (Recommended)**

| Hardware | Role | Notes |
|---|---|---|
| BGP-X Router v1 | Full bridge + exit (optional) | Best for home/community gateway use |
| BGP-X Node v1 | Bridge or relay | Best for outdoor/mast community deployment |
| BGP-X Gateway v1 | Provider-grade bridge/exit | Best for high-throughput public infrastructure |

These devices have integrated LoRa (SX1262) and WiFi (802.11s), plus WAN Ethernet, in a single enclosure.

**Option B: Third-Party Router + USB Adapter**

A compatible OpenWrt router (e.g., GL.iNet GL-MT6000, Banana Pi BPi-R3) with a BGP-X Adapter/Dongle (USB LoRa modem) can serve as a domain bridge node. The router handles the bgpx-node daemon and clearnet connection; the adapter provides the mesh transport.

**Minimum Hardware Capability**: The device must run the full `bgpx-node` daemon (requires Linux, sufficient RAM/CPU). Tier 2 Client Nodes (LILYGO T3, etc.) cannot serve as bridge nodes — they lack the routing stack.

### 2.2 Clearnet WAN Options

Any reliable internet connection serves as the clearnet side of the bridge:

- **Fiber / Cable / DSL**: Standard ISP connection
- **Cellular (4G/5G)**: Via USB modem or integrated cellular
- **Satellite internet (Starlink, Iridium, Inmarsat, etc.)**: Treated as **clearnet domain (0x00000001)**.

### 2.3 Mesh Radio Options

| Radio Type | Range | Latency | Notes |
|---|---|---|---|
| WiFi 802.11s | 100m–2km (LOS) | 5–20ms | Higher bandwidth; best for neighborhood coverage |
| LoRa (SF7–SF12) | 2–15km (LOS) | 100ms–5s | Long range; best for wide-area sparse coverage |
| BLE | 10–100m | Low | Short-range; best for indoor/dense urban |

### 2.4 For High-Traffic Bridge Nodes

If your bridge serves a large island or high-volume public traffic:

- BGP-X Router v1 with 512 MB RAM (Extended configuration)
- BGP-X Gateway v1 (for provider-grade capacity)
- Stable WAN with high uptime ISP connection
- Elevated antenna for LoRa range
- Monitor RAM usage: bridge nodes maintain two path_id tables (single-domain and cross-domain). At 10,000 concurrent cross-domain paths: ~2.5 MB overhead. High-traffic bridges should provision 1–2 GB RAM.

---

## 3. Configuration

### 3.1 Basic Domain Bridge Configuration

```toml
[node]
role = "relay,domain_bridge"
bridge_capable = true           # Automatically set if 2+ routing_domains configured

[[routing_domains]]
domain_type = "clearnet"
endpoints = [
  { protocol = "udp", address = "0.0.0.0", port = 7474 }
]

[[routing_domains]]
domain_type = "mesh"
island_id = "your-island-id"
transports = ["lora", "wifi-mesh"]
wifi_interface = "mesh0"        # If WiFi mesh configured

# Bridge pair is auto-generated from routing_domains:
# bridges = [
#   { from = "clearnet", to = "mesh:your-island-id",
#     bridge_latency_ms = 200, available = true }
# ]
```

### 3.2 Setting Accurate Bridge Latency

The `bridge_latency_ms` value tells path construction algorithms how much additional latency crossing your bridge adds. Set it accurately:

- WiFi 802.11s bridge: 5-20ms
- LoRa SF7 bridge: 100-300ms
- LoRa SF9 bridge: 300-600ms
- LoRa SF12 bridge: 800-2000ms

Deliberately underreporting latency results in `BridgeLatencyExceeded` reputation events. Be accurate.

```bash
# Measure your actual bridge latency
bgpx-cli domains bridges --latency-report
# Adjust bridge_latency_ms in config to match measured values
```

### 3.3 Configuring Multiple Bridge Pairs

If your node has multiple mesh radios (e.g., two LoRa interfaces on different channels for different islands):

```toml
[[routing_domains]]
domain_type = "mesh"
island_id = "island-north"
transports = ["lora"]
lora_channel = 0               # Different LoRa channel per island

[[routing_domains]]
domain_type = "mesh"
island_id = "island-south"
transports = ["lora"]
lora_channel = 1               # Different channel
```

Each island gets its own bridge pair advertisement. Path construction treats each pair independently.

---

## 4. Publishing Your Bridge to the DHT

Your bridge advertisement publishes automatically when the daemon starts. The advertisement is stored in the **unified DHT** — a single DHT spanning all routing domains. Mesh-only nodes access the unified DHT via your bridge node's cache.

Verify it's published:

```bash
# Check bridge advertisement freshness
bgpx-cli domains bridges --self --dht-freshness
# Expected: bridge record age < 24 hours (8-hour republication interval)

# Check bridge is discoverable by others
bgpx-cli domains bridges --self
# Expected: bridge pairs listed with from/to domains, latency, available=true
```

---

## 5. Monitoring Bridge Health

### 5.1 Daily Checks

```bash
bgpx-cli domains bridges --health
```

Output:
```
Bridge pairs:
  clearnet → mesh:your-island-id
    Status: ONLINE
    Advertised latency: 300ms
    Measured latency (p50): 285ms ✓
    Measured latency (p95): 420ms
    Transitions (24h): 1,247
    Bridge available: true
    DHT record age: 6 hours (fresh)
```

### 5.2 What to Watch

| Metric | Warning | Action |
|---|---|---|
| Bridge available | false | Investigate transport issue; fix and re-enable |
| Measured latency > 200% of advertised | Sustained | Update advertised latency in config |
| Transitions = 0 for hours | Suspicious | Check if bridge is being selected; check reputation |
| DHT record age > 20 hours | Warning | Check daemon is running and republishing |

### 5.3 Reputation Events for Bridge Nodes

Your node earns reputation points for reliable bridge operation:

- `DomainBridgeSuccess` (+0.5): successful cross-domain path
- `BridgeLatencyWithinAdvertised` (+0.3): latency within advertised range
- `DomainBridgeFailure` (-3.0): bridge failed
- `BridgeLatencyExceeded` (-1.0): latency > 200% of advertised
- `BridgeClaimedAvailableOffline` (-4.0): advertised available but couldn't deliver

Keep your reputation score above 0.6 to maintain good selection weight in path construction.

---

## 6. Temporarily Taking a Bridge Offline

For planned maintenance, mark your bridge as unavailable before starting maintenance:

```bash
# Mark bridge pair unavailable (stops new paths from using this bridge)
bgpx-cli domains bridge-availability \
    --from clearnet \
    --to mesh:your-island-id \
    --available false

# Existing sessions will KEEPALIVE_TIMEOUT and rebuild via other bridges
# Wait 2 minutes for sessions to migrate

# Perform maintenance

# Re-enable when done
bgpx-cli domains bridge-availability \
    --from clearnet \
    --to mesh:your-island-id \
    --available true
```

Marking unavailable properly prevents `BridgeClaimedAvailableOffline` reputation penalties and allows sessions to migrate gracefully.

---

## 7. For Mesh Island Gateways Specifically

If you're the gateway for a mesh island (the node that bridges clearnet to LoRa for a community):

### 7.1 Island Advertisement

```bash
# Verify island is published to unified DHT
bgpx-cli islands show --self --dht-freshness
# Expected: island record present; fresh

# If not published, force publication:
bgpx-cli islands publish your-island-id
```

### 7.2 Minimum Standards for Island Gateways

- **Uptime**: aim for >99% (island members depend on you for internet access)
- **Redundancy**: coordinate with at least one other operator to run a second gateway for your island
- **WAN monitoring**: alert if WAN goes down; restore within 1 hour if possible
- **Antenna**: ensure LoRa antenna is at sufficient height for island coverage

### 7.3 Emergency Island Isolation

If your island experiences a security incident (rogue node, protocol violation), remove the island from the DHT to stop new cross-domain paths:

```bash
bgpx-cli islands withdraw your-island-id
# Prevents new cross-domain paths to the island
# Intra-island traffic continues (cannot be stopped from gateway)
# Re-publish when incident resolved:
bgpx-cli islands publish your-island-id
```

---

## 8. Geographic Plausibility (Optional)

Geographic plausibility scoring is an **OPTIONAL** reputation signal. Nodes are NOT required to declare a jurisdiction.

If you choose to declare a jurisdiction in your node advertisement:
- Geo plausibility scoring will apply
- RTT measurements will be compared to expected ranges for your declared region
- Implausible scores generate reputation penalties (not hard blocks)

To declare a jurisdiction:

```toml
[node]
# ... other config ...
jurisdiction = "US-CA"  # Optional: ISO 3166-2 country/region code
```

**Satellite-connected nodes are exempt** from geo plausibility scoring because satellite terminal IPs may be assigned from distant ground stations. If your bridge uses Starlink WAN and declares a jurisdiction, geo plausibility scoring is automatically bypassed for the clearnet side.

---

## 9. Satellite WAN as Clearnet

If your bridge uses a satellite internet service (Starlink, Iridium, Inmarsat, etc.) as its WAN:

- **Domain**: Your WAN connection is clearnet (0x00000001), not a separate "satellite domain"
- **Latency class**: High latency (20–600ms+ depending on orbit)
- **Configuration**: No special BGP-X configuration needed; satellite is just another clearnet transport
- **Path construction**: Clients will select your bridge knowing the higher latency (advertise it accurately in `bridge_latency_ms`)

**Domain type 0x00000005 is reserved** for a future BGP-X-native satellite network where satellites run BGP-X software directly. It is NOT for current commercial satellite internet. Your satellite-connected bridge is a clearnet-to-mesh bridge.

---

## 10. Legal Considerations

As a domain bridge operator, you facilitate traffic between routing domains. Key points:

- Your bridge sees only encrypted onion layers (except your own DOMAIN_BRIDGE layer) — you cannot read the content of bridged traffic
- You cannot determine who is communicating with whom through your bridge
- You are similar in legal position to an internet exchange point or transit provider
- Review `legal/liability.md` for jurisdiction-specific considerations
- Consult legal counsel before operating bridge nodes in jurisdictions with uncertain intermediary liability law

---

## 11. Troubleshooting

### Bridge Shows Offline But WAN and Mesh Are Up

1. **Check DHT connectivity**:
   ```bash
   bgpx-cli domains bridges --dht-freshness
   ```
   If record is stale, check if your node is participating in the DHT:
   ```bash
   bgpx-cli node dht-stats
   ```

2. **Check bridge pair status**:
   ```bash
   bgpx-cli domains bridges --self
   ```
   Look for `available = false` in output.

3. **Force republication**:
   ```bash
   bgpx-cli domains bridges --self --force-refresh
   ```

### High Reputation Penalty (BridgeLatencyExceeded)

Your advertised latency is significantly lower than actual. Measure real latency and update your configuration:

```bash
bgpx-cli domains bridges --latency-report
# Edit config: bridge_latency_ms = <measured_p95>
systemctl restart bgpx-node  # or appropriate restart command
```

### Bridge Not Being Selected

If your bridge has good uptime but path construction isn't selecting it:

1. Check reputation score:
   ```bash
   bgpx-cli reputation show --self
   ```

2. Check latency accuracy (advertising too high latency reduces selection probability for latency-sensitive traffic)

3. Check if bridge is marked available:
   ```bash
   bgpx-cli domains bridges --self
   ```

### WAN Failover (If Using Backup Connection)

If your primary WAN fails and you have a backup (e.g., cellular):

1. The bridge will show degraded status if WAN latency changes significantly
2. Update `bridge_latency_ms` if latency class changes dramatically (e.g., fiber → satellite)
3. Monitor for `BridgeLatencyExceeded` reputation events during failover

---

## 12. Next Steps

- **Monitor continuously**: Set up alerts for `Bridge available = false` and `DHT record age > 20 hours`
- **Coordinate with other operators**: Mesh islands should have at least two independent gateways
- **Keep firmware updated**: Security updates are critical for infrastructure nodes
- **Review logs**: Check for `DomainBridgeFailure` events indicating transport issues
