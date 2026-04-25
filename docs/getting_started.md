# Getting Started with BGP-X

This document is your entry point to understanding, deploying, and using BGP-X. It is written for four audiences:

- **Network operators** — people who want to deploy BGP-X on a router for their network
- **Node operators** — people who want to run BGP-X relay, entry, or exit nodes
- **Developers** — people who want to build applications using the BGP-X SDK
- **End users** — people whose traffic is protected by a BGP-X router

---

## Deployment Modes

BGP-X supports six deployment modes. Choose the one that fits your situation:

| Mode | Description | ISP Needed | Use Case |
|---|---|---|---|
| Dual-Stack Router | BGP + BGP-X coexist; routing policy selects per flow | Yes | Privacy with selective bypass for gaming, streaming, etc. |
| BGP-X Only Router | All LAN traffic through overlay; no bypass | Yes (as transport) | Maximum privacy enforcement |
| Standalone Device | BGP-X daemon on one device only | Yes | No router access; travel; development |
| Mesh Node | No ISP; mesh radio transport only | No | Community without ISP; disaster recovery |
| Gateway Node | Bridges mesh to clearnet internet | Yes (at gateway) | Shared internet access for mesh community |
| Broadcast Amplifier | Range extension only; no routing | No | Extending mesh coverage |

**For most households**: Dual-Stack Router or BGP-X Only Router.

**For communities without reliable internet**: Mesh Node + Gateway Node.

**For developers**: Standalone Device or SDK with Embedded Mode.

---

## What BGP-X Provides

BGP-X provides a **network-layer privacy overlay**. Concretely:

When your traffic routes through BGP-X:

- Your ISP sees only encrypted traffic to the first BGP-X relay — not your destination
- The entry node knows your IP address but not your destination
- The exit node knows the destination but not your IP address
- No single party in the chain can see both ends simultaneously
- All traffic between hops is encrypted with per-hop, per-session keys

BGP-X does not make you invisible. It distributes and limits what any single party can observe.

---

## Architecture at a Glance

BGP-X is router infrastructure — it runs on your router and protects all connected devices:

```
[Phone]────┐
[Laptop]───┼──► [BGP-X Router] ──► [BGP-X Overlay Network] ──► [Internet]
[TV]───────┘          │
                      │  Routing Policy Engine decides per flow:
                      ├──► BGP-X overlay (privacy-protected)
                      └──► Standard routing (direct, unprotected)
```

All devices connect to the router as normal. The BGP-X daemon handles everything transparently.

---

## Who Uses What

| You Are | What You Need | Configuration |
|---|---|---|
| Device on LAN | Nothing — just connect to router | Zero |
| Building a BGP-X native app | BGP-X SDK | SDK integration |
| Managing the router | bgpx-cli or GUI | Control socket access |
| Running a relay node | bgpx-node | Node configuration |
| Running an exit gateway | bgpx-node + exit policy | Exit configuration + DoH/ECH |

---

## Dual-Stack vs. BGP-X Only

**Dual-Stack** (recommended for most):
- Some traffic goes through BGP-X (private)
- Some traffic goes direct (unprotected, lower latency)
- Routing policy decides which traffic is which
- Good for: households where some devices need low latency (gaming, smart TV)

**BGP-X Only** (maximum privacy):
- All LAN traffic forced through overlay
- No bypass possible from LAN devices
- Slightly higher latency for all traffic
- Good for: activist networks, high-security environments, shared community networks

---

## For LAN Devices — No Configuration Needed

If your router runs BGP-X, every device on your network is protected without any changes:

- Connect to the router's WiFi or Ethernet normally
- Your traffic is automatically routed through BGP-X (for traffic matching the routing policy)
- No app changes, no VPN client, no browser extensions

---

## For Developers — SDK Connection Model

The BGP-X SDK does NOT implement its own routing. It connects to the daemon:

```
Your App → SDK → Unix Socket → BGP-X Router Daemon → Overlay Network
```

The daemon handles all path construction, encryption, and DHT. Your app just calls `connect_stream()` and gets back a socket.

If the daemon is on a router and your app is on a LAN device:
- SSH socket forwarding: `ssh -L /tmp/bgpx.sock:/var/run/bgpx/sdk.sock user@router`
- TCP endpoint: router exposes `192.168.1.1:7475` on LAN (requires authentication)
- Or use SDK Embedded Mode (full daemon in-process, for mobile)

---

## DHT Pools

Pools are named, signed collections of BGP-X nodes grouped by trust level. Instead of one flat pool of all relays, you can:

- Use the **default pool** for general traffic
- Use a **curated pool** (signed by a trusted curator) for higher trust
- Use a **private pool** (your own nodes) for exit infrastructure you control
- Chain segments from different pools for multi-tier trust paths

Example: `Entry from default pool → Relay from trusted pool → Exit from my private pool`

This is the **double-exit architecture** — two independent exit pools in sequence:

```
Client → [default pool entry + relay] → [private pool exit 1] → [private pool exit 2] → Destination
Exit 1 sees: traffic going to Exit 2 (not destination)
Exit 2 sees: traffic from Exit 1 (not client)
```

---

## Encrypted Client Hello (ECH)

When you connect to a clearnet destination through a BGP-X exit node, the exit node must connect to the destination via TLS. Normally, TLS ClientHello includes the Server Name Indication (SNI) — the domain name in plaintext.

ECH (Encrypted Client Hello) hides the domain name from SNI when:
- The destination publishes an ECH configuration in its DNS HTTPS record
- The exit node is ECH-capable (`ech_capable = true` in exit policy)

Result: even the exit node does not see the domain name in plaintext. The network sees only the destination IP.

---

## Mesh Modes

BGP-X supports operating without any ISP connection:

**Mesh Node (Mode 4)**: runs on a router with no WAN connection. Communicates with other BGP-X nodes via WiFi 802.11s mesh or LoRa radio. Can be deployed in rural communities, for disaster recovery, or anywhere ISP access is unavailable.

**Gateway Node (Mode 5)**: has both mesh radio and a WAN connection. Acts as the bridge — mesh users route clearnet traffic through the gateway. The gateway's ISP sees the traffic; individual mesh users have no direct ISP exposure.

**Broadcast Amplifier (Mode 6)**: minimal hardware (no Linux required), single radio. Receives and rebroadcasts BGP-X mesh packets to extend range. No routing intelligence.

---

## Pluggable Transport

Your ISP can observe that you're connecting to BGP-X entry nodes (encrypted UDP on port 7474). Pluggable transport obfuscates this traffic pattern to look like something else.

BGP-X's built-in PT uses obfs4-style obfuscation (random padding + stream cipher). External PT subprocess interface is also supported.

Enable with: `[pluggable_transport] enabled = true` in daemon config.

---

## What BGP-X Does Not Do

Be clear about this before deploying:

- **BGP-X does not protect you if your application leaks your identity.** Logging into a service with your real username tells the service who you are.
- **BGP-X does not encrypt content beyond the overlay.** Always use HTTPS for clearnet connections — the exit node sees destination IP and can see plaintext HTTP content.
- **BGP-X does not protect you if your device is compromised.** Malware can observe your traffic before encryption.
- **BGP-X does not hide that you use BGP-X from your ISP.** Your ISP sees encrypted UDP to an entry node. Pluggable transport mitigates this.
- **BGP-X does not prevent traffic blocking.** Your ISP can block port 7474. Pluggable transport mitigates this.
- **BGP-X does not make illegal activities legal.** Using BGP-X does not grant legal protection for any conduct.

---

## Security Recommendations

1. Always use HTTPS for clearnet connections
2. Enable ECH-requiring exit constraints for sensitive destinations
3. Use private pool for exit if you control server infrastructure
4. Use pluggable transport if you're in an environment that blocks or monitors VPN/proxy traffic
5. Use BGP-X-only firmware for maximum enforcement
6. Verify node signatures — never disable signature verification
7. If operating an exit gateway: publish no-log policy, use DoH with DNSSEC and ECS stripping, enable ECH

---

## Next Steps

- **Understand the architecture**: `ARCHITECTURE.md` and `docs/deployment_architecture.md`
- **Understand BGP/BGP-X coexistence**: `docs/bgp_bgpx_coexistence.md`
- **Deploy on a router**: `/deployment/node_setup.md`
- **Deploy a mesh network**: `/deployment/mesh_deployment.md`
- **Build an application**: `/docs/application_guide.md` and `/sdk/sdk_spec.md`
- **Run a relay or exit node**: `/node/node.md`
- **Understand the threat model**: `/security/threat_model.md`
- **Understand pools**: `/protocol/pool_spec.md`
