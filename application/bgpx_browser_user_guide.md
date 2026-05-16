# BGP-X Browser User Guide

**Version**: 0.1.0-draft
**License**: CC BY 4.0 (Documentation)
**Target Application**: BGP-X Browser (Desktop, Android, iOS)

---

## 1. What is BGP-X Browser?

BGP-X Browser is a privacy-focused web browser that connects to the BGP-X network. It is designed to be the primary interface for users interacting with the BGP-X ecosystem, serving two core functions that standard browsers cannot:

1.  **Browse `.bgpx` sites** — Private, self-authenticating services hosted within the BGP-X overlay network, similar to how Tor Browser allows browsing `.onion` sites.
2.  **Route all your web traffic through the BGP-X overlay** — Your IP address is hidden from websites you visit; your ISP cannot see your browsing destinations.

Conceptually, it is a fusion of the privacy guarantees of Tor Browser with the accessibility of Firefox/Chrome, but built specifically for the BGP-X architecture. You can browse standard websites privately AND access BGP-X-native services hosted anywhere—on clearnet servers, in mesh islands, or satellite-connected nodes.

---

## 2. Installation

### Linux

**AppImage** (Recommended — works on any modern Linux distribution):
```bash
chmod +x BGP-X-Browser-0.1.0.AppImage
./BGP-X-Browser-0.1.0.AppImage
```

**Debian/Ubuntu (.deb)**:
```bash
sudo dpkg -i bgpx-browser_0.1.0_amd64.deb
# Launches from: Applications → Internet → BGP-X Browser
```

### macOS

1.  Download `BGP-X-Browser-0.1.0.dmg`
2.  Open the .dmg and drag BGP-X Browser to your Applications folder
3.  **First Launch**: Right-click the app icon and select "Open" (required by macOS security for unsigned applications on first run)

### Windows

1.  Download `BGP-X-Browser-Setup-0.1.0.exe`
2.  Run the installer and follow the setup wizard
3.  BGP-X Browser will appear in your Start Menu

### Android & iOS

Mobile applications for Android and iOS are available separately. Search for "BGP-X Browser" on the Google Play Store or Apple App Store. The mobile version connects to an embedded BGP-X daemon on your device or to a local BGP-X Router.

---

## 3. Connecting to Your BGP-X Router

BGP-X Browser needs to connect to a running `bgpx-node` daemon to function. The daemon handles all the routing logic, encryption, and DHT queries—the browser serves as the user interface and network stack.

### Scenario A: You Have a BGP-X Router v1

The BGP-X Router v1 runs the daemon automatically at startup. BGP-X Browser on any device on your LAN will typically find it via mDNS/zeroconf auto-discovery.

**If auto-connect fails:**
1.  Open BGP-X Browser settings (gear icon → **Settings** → **Connection**)
2.  Set the daemon address to your router's LAN IP: `192.168.1.1:7475` (or the IP/port configured in your router's LuCI interface)
3.  Click **Connect**

### Scenario B: You Are Using a BGP-X Client Node (Tier 2)

The BGP-X Client Node is a low-cost, battery-powered hardware device designed for individual users. It broadcasts a WiFi network that your phone or laptop can connect to.

1.  Connect your device to the Client Node's WiFi network (default SSID usually `BGPX-Client-XXXX`)
2.  Open BGP-X Browser. It will detect the daemon running on the gateway IP (typically `192.168.2.1` or similar)
3.  The Client Node routes your traffic through its LoRa or WiFi mesh connection

**Note**: The Client Node runs subset firmware. It supports client connectivity but may not support advanced features like full DHT storage or high-bandwidth relay for other users.

### Scenario C: You Have a BGP-X Adapter/Dongle (Tier 3)

The BGP-X Adapter is a USB LoRa modem. To use the browser with it:

1.  Plug the adapter into your computer
2.  Install and configure the `bgpx-node` daemon on your computer to use the USB serial port as a LoRa transport (see `docs/user_guide/standalone_device.md`)
3.  Launch BGP-X Browser. It will detect the local daemon.

### Scenario D: Standalone Desktop/Server

If you installed `bgpx-node` directly on your computer (Linux, macOS, Windows), BGP-X Browser will find the local daemon at `127.0.0.1` or `/var/run/bgpx/sdk.sock` automatically.

### Connection Status

The status bar at the bottom of the browser indicates connectivity:

-   🟢 **Connected** — Browser is connected to BGP-X daemon; all traffic is routed through the overlay.
-   🔴 **Disconnected** — No daemon found. The browser may still function, but traffic is NOT private.

---

## 4. Browsing .bgpx Sites

### Entering a .bgpx Address

You can enter a `.bgpx` address directly in the address bar. The browser recognizes both human-readable short names and full cryptographic addresses:

```
community-news.bgpx
bgpx://library.bgpx/books
bgpx://a3f2b9c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1.bgpx/
```

-   **Short name** (e.g., `community-news.bgpx`): Resolved via the BGP-X Name Registry DHT.
-   **Full address** (64 hex characters): The service's actual cryptographic identity (ServiceID).

### What Happens When You Visit a .bgpx Site

1.  **Resolution**: The browser queries the BGP-X daemon to resolve the name (if a short name) to a ServiceID.
2.  **Path Construction**: The daemon constructs a private path through the BGP-X network to the service. This path may traverse clearnet relays, mesh islands, or satellite links.
3.  **Onion Routing**: Your request travels through multiple relay nodes. No single node knows both who you are and what you're looking for.
4.  **Delivery**: The site receives your request and responds. The response travels back through the same path.
5.  **Verification**: The browser verifies the site's identity using the ServiceID. The green `.bgpx` badge confirms the site is authentic—no certificate authority required. The address *is* the identity.

### If the Site Is Slow (LoRa or Satellite Path)

Some `.bgpx` sites are hosted inside mesh radio networks (LoRa islands) or are connected via satellite backhaul. Reaching them involves radio signals that are inherently slower than fiber.

The browser shows a latency indicator:
```
[🟢 community-news.bgpx]  [⏸ LoRa ~3s/resource]
```
```
[🟢 news-link.bgpx]       [🛰 Satellite ~2s/resource]
```

**Satellite Note**: Satellite internet (Starlink, Iridium, Inmarsat) is treated as **clearnet** by BGP-X. A site hosted on a server with Starlink WAN is a clearnet `.bgpx` service. A site hosted *inside* a mesh island with a satellite gateway involves the "Satellite" transport indicator.

**LoRa Path Behavior**:
-   **Expected latency**: 3-15 seconds per page.
-   **Progressive Loading**: The browser uses HTTP/2 multiplexing to fetch multiple resources in parallel over a single stream. It shows the page skeleton (layout) immediately while resources load.
-   **Never Blank**: You will never see a blank white page. There is always something visible.

You can interact with partially-loaded pages. Think of it like a slow, reliable connection—it works, but requires patience.

---

## 5. Understanding the Address Bar

```
[🟢 .bgpx] community-news.bgpx/article/1234    [⏸ LoRa ~3s/resource]
```

**Left Badge** (Security Indicator):
-   🟢 **.bgpx** — Verified BGP-X native service. Connection is private; site identity is cryptographically confirmed.
-   🌐 **Globe** — Standard clearnet website. Traffic is routed through the BGP-X overlay (your IP is hidden), but the site uses standard HTTPS.
-   🔶 **Warning** — Clearnet website NOT going through the overlay. Check your daemon connection or routing policy.
-   🔴 **Error** — Site identity could not be verified. Do not trust this site.

**Center** — The address you are visiting.

**Right** (Latency Indicator — shown only for high-latency paths):
-   `[⏸ LoRa ~3s/resource]` — Path traverses a LoRa radio segment.
-   `[🛰 Satellite ~2s/resource]` — Path traverses a satellite segment (e.g., mesh island gateway uses Starlink).

**Clicking the Left Badge** reveals a connection summary: path type, hop count, domains traversed, and whether Encrypted Client Hello (ECH) was used at the exit.

---

## 6. Privacy Features

### Your IP Address is Hidden

When browsing through BGP-X Browser with a connected daemon, websites see the IP address of the BGP-X exit node—not your real IP. The exit node does not know who you are. If the site is a `.bgpx` native service, it sees only the last relay's identity, never the client.

### BGP-X-Specific Protections

BGP-X Browser includes these protections by default:
-   **Ad and Tracker Blocking**: Built-in; no extension needed.
-   **Anti-Fingerprinting**: Prevents websites from identifying you by browser/hardware details.
-   **Cookie Isolation**: Each `.bgpx` site gets its own separate cookie jar. Sites cannot track you across `.bgpx` services.
-   **HTTPS-Only**: Clearnet sites are always loaded over HTTPS.
-   **No Clearnet DNS Leakage**: DNS queries go through the BGP-X overlay.
-   **ECH (Encrypted Client Hello)**: When connecting to clearnet sites that support ECH, BGP-X exit nodes use it to hide the domain name (SNI) from the exit node itself and network observers.

### What BGP-X Browser Cannot Protect Against

-   **Your Own Actions**: If you log into a website with your real username, the site knows who you are.
-   **Malicious .bgpx Sites**: BGP-X confirms a site's identity (address matches key), but cannot tell you if the site operator is trustworthy.
-   **Device Compromise**: If your device has malware, BGP-X Browser cannot protect you.

---

## 7. .bgpx Mixed Content Warning

If a `.bgpx` page attempts to load content from regular clearnet websites (images, scripts), you will see a notice:

```
⚠️ This .bgpx page loads content from clearnet websites.
   Those servers may log your activity.
   [Block clearnet content] [Continue] [More info]
```

-   **Block clearnet content**: Keeps your session fully within the BGP-X network. May break some page features.
-   **Continue**: Loads clearnet content through the BGP-X overlay (your IP is still protected from the clearnet server, but the server knows something accessed the page).

---

## 8. Settings Reference

**Settings → Connection**:
-   **Daemon Address**: Auto-detect or manual IP/Port.
-   **Pool Selection**: Choose which relay pools the daemon uses for path construction.
-   **Cover Traffic**: Enable/disable. Adds dummy traffic to make timing analysis harder. Increases bandwidth usage.

**Settings → Privacy**:
-   **Block Clearnet Sub-resources on .bgpx pages**: Default is "Ask".
-   **Anti-Fingerprinting Level**: Standard / Strict / Maximum.
-   **Cookie Isolation**: Recommended is "Per-ServiceID" for `.bgpx`, "Per-Site" for clearnet.

**Settings → Performance**:
-   **Cache Size**: Larger cache = faster repeat visits, more disk usage.
-   **LoRa Batching**: Controls how aggressively the browser batches requests. "Aggressive" reduces round-trips but uses more memory.
-   **Preloading**: Load likely next pages in background. Costs LoRa bandwidth.

**Settings → Trusted Pools** (Advanced):
-   Add private relay pools for routing through servers you or your community operate.
-   Configure pools specifically for exits in preferred jurisdictions (e.g., choose an EU-based exit pool for GDPR considerations).

---

## 9. Troubleshooting

### "BGP-X Not Running"

The browser cannot find the daemon.

**Fixes**:
-   If using **BGP-X Router v1**: Ensure the router is powered on and connected.
-   If using **bgpx-node on your computer**: Run `systemctl start bgpx-node` (Linux) or start the service via your init system.
-   Check **Settings → Connection** and try entering the daemon address manually.

### "Service Offline"

The `.bgpx` service could not be reached.

**Possible Causes**:
-   The service is genuinely offline.
-   The service's mesh island is unreachable (gateway node may be offline).
-   You mistyped the address.
-   No domain bridge node exists between your location and the service's mesh island.

### "LoRa Path Paused"

The LoRa radio on a relay node has hit its regulatory duty cycle limit (e.g., 1% in EU). This is temporary. The browser shows a countdown. Wait for the duty cycle to reset, or click **Cancel** to try constructing a different path.

### Page Loads Very Slowly

-   **If on a LoRa path**: This is expected. See Section 4.
-   **If on a clearnet path**: Check the hop count in Connection Info. High hop counts add latency. Check if the site itself is slow.

### .bgpx Short Name Not Found

The short name may not be registered, expired, or the Name Registry DHT may be unreachable. Try the full 64-character `.bgpx` address if you know it.

---

## 10. Frequently Asked Questions

**Is BGP-X Browser based on Firefox or Chrome?**
It uses a standard rendering engine (Chromium or Gecko) with a BGP-X-specific networking backend. This allows it to route traffic through the BGP-X daemon while retaining familiar browser behavior.

**Can I use extensions?**
A limited set of extensions compatible with the modified networking stack are supported. Standard extensions that assume standard TCP/IP connectivity may not function correctly when routing through the overlay. The built-in content blocker covers most privacy needs.

**Does my ISP know I'm using BGP-X Browser?**
Your ISP can see you are connecting to BGP-X entry nodes (encrypted UDP on port 7474). They cannot see the destination or content. If you need to hide BGP-X participation, enable Pluggable Transport in the daemon settings.

**Can I use BGP-X Browser and Tor Browser at the same time?**
Yes. They are independent applications. Running both routes their traffic through separate networks.

**Does BGP-X Browser work on mobile?**
Yes. Separate apps for Android and iOS are available. They operate in "Embedded Mode" (running the daemon in-process) or connect to a local Router/Client Node.

**What hardware do I need to use BGP-X Browser?**
You only need a computer or phone. To access the BGP-X network, you need one of the following:
-   A **BGP-X Router v1** on your LAN.
-   A **BGP-X Client Node** (Tier 2 hardware) that creates a WiFi hotspot.
-   A **BGP-X Adapter** (Tier 3 USB dongle) connected to your computer.
-   A standalone installation of the `bgpx-node` daemon on your device.
