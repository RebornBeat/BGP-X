# BGP-X Router v1 Setup Guide

**Version**: 0.1.0-draft
**Target Hardware**: BGP-X Router v1 (All-in-One)

---

## 1. What You Have

The **BGP-X Router v1** is an all-in-one router and full BGP-X routing node. It replaces your existing home or office router entirely and automatically protects all devices on your network. Out of the box, it provides:

- **Standard router functions**: WAN connection, DHCP, NAT, firewall, WiFi access point for your devices
- **BGP-X overlay routing**: all internet traffic from your LAN is transparently routed through the privacy overlay
- **LoRa radio (sub-GHz)**: participates in local mesh islands
- **WiFi 802.11s mesh**: connects to other BGP-X mesh nodes via WiFi backhaul
- **Bluetooth BLE**: optional BLE mesh transport
- **Domain bridge capability**: bridges mesh islands to clearnet internet if configured
- **Hardware security**: TPM 2.0 chip stores your node's cryptographic identity

**No technical experience required for basic setup.** The setup wizard handles standard ISP connections, WiFi network creation, and initial BGP-X configuration.

**Note**: If you have different hardware (BGP-X Node v1, Client Node, Adapter, or OpenWrt package), refer to the specific guides in the `/docs/user_guide/` directory.

---

## 2. What's in the Box

- **BGP-X Router v1 unit** (desktop carrier or outdoor IP67 carrier depending on SKU)
- **Power supply** (12V DC barrel jack) **OR** PoE injector/cable (depending on carrier variant)
- **Ethernet cables** (4× short patch cables)
- **LoRa antenna** (rubber duck type, SMA male; frequency matches your region's SKU: 433/868/915 MHz)
- **WiFi antennas** (4× dipole antennas, RP-SMA)
- **Quick start card** (basic connection diagram and default access info)

**If outdoor variant**: Also includes mast/pole mounting brackets and weatherproofing gaskets.

---

## 3. Physical Setup

**Do this before powering on.**

### 3.1 Antenna Installation (Critical)

1.  Attach the **LoRa antenna** to the SMA connector labeled **LoRa**. Tighten finger-tight, then an additional 1/4 turn with a wrench if necessary. **Do not overtighten.**
2.  Attach the **four WiFi antennas** to the RP-SMA connectors labeled **WiFi**. Finger-tight plus 1/4 turn.
3.  Position all antennas upright (vertical orientation) for best omni-directional signal.

**WARNING: Never power on without antennas attached.** Transmitting without a load can damage the radio power amplifiers.

### 3.2 Connecting to Your ISP (WAN)

1.  Locate the Ethernet cable coming from your ISP's modem, ONT (fiber box), or satellite terminal.
2.  Plug it into the **WAN** port on the BGP-X Router v1 (labeled **WAN** or port **0**).
3.  If replacing an existing router, unplug the old router from that modem first.

**Satellite Modem (Optional)**:
If you are using a USB satellite modem (Starlink Gen 3, Iridium Certus, etc.):
- Connect the modem to one of the **USB 3.0 ports** on the router.
- The router will auto-detect the USB vendor ID and configure the connection as a clearnet WAN interface with satellite-class latency (20-600ms RTT). **Note:** From BGP-X's perspective, satellite internet is **clearnet domain**, not a separate routing domain.

### 3.3 Connecting LAN Devices

- Plug Ethernet cables from the **LAN ports** (labeled 1, 2, 3...) to any wired devices (desktop PCs, smart TVs, game consoles).
- WiFi devices will connect wirelessly after the WiFi network is configured in the wizard.

### 3.4 Power On

1.  Connect the power supply or PoE cable.
2.  Wait **30-60 seconds** for the device to boot.
3.  Observe the **Status LED**. It should turn solid green when ready.
    - **Boot sequence**: Red (startup) → Amber (loading) → Solid Green (ready).
    - **If blinking amber**: BGP-X daemon is initializing.
    - **If fast blinking amber**: BGP-X error. Check logs via web interface.

---

## 4. First-Time Setup Wizard

### 4.1 Accessing the Setup Wizard

Connect a computer or phone to the BGP-X Router v1:

- **Wired**: Plug an Ethernet cable into any LAN port.
- **Wireless**: Connect to the default WiFi network `BGP-X-Setup` (default password: `bgpxsetup`).

Open a web browser and navigate to: `http://192.168.1.1`

The **BGP-X Setup Wizard** launches automatically.

### 4.2 Step 1: WAN Configuration

Select how your ISP provides internet connectivity:

| Option | Description |
|---|---|
| **DHCP** (most common) | Router automatically obtains an IP address. Select this if your ISP doesn't require a login. |
| **PPPoE** | Requires username and password from your ISP. Common with DSL providers. |
| **Static IP** | Your ISP provided a fixed IP address, gateway, and DNS servers. |
| **USB Satellite** | Configures an attached USB satellite modem as the WAN interface. |

Click **Test Connection**. If successful, you will see your public IP address.

**Important Note on Satellite WAN**:
If using satellite internet (Starlink, Iridium, Inmarsat, HughesNet, Viasat), be aware that BGP-X treats this as **clearnet domain** (domain type `0x00000001`), the same as fiber or cellular. The only difference is the higher latency. All BGP-X privacy protections apply equally. Domain type `0x00000005` (bgpx-satellite) is reserved for future BGP-X-native satellite networks and is not currently used.

### 4.3 Step 2: Admin Password

Set a strong password for the router admin interface (`http://192.168.1.1`). This protects your router's configuration from unauthorized changes.

**Write this down and store it securely.** You cannot recover it without a factory reset.

### 4.4 Step 3: WiFi Configuration

Configure your primary WiFi network (SSID):

- **Network Name (SSID)**: The name your devices will see.
- **Password**: WPA3-Personal (recommended) or WPA2-PSK.
- **Separate Bands (Optional)**: You may configure different SSIDs for 2.4 GHz and 5 GHz bands if desired.

This WiFi network is for your standard LAN traffic, which will be automatically routed through the BGP-X overlay by default.

### 4.5 Step 4: BGP-X Node Role

Choose what role this router plays in the BGP-X network:

**Option A: Relay Only (Default)**
- **What it does**: Your router participates in the BGP-X overlay as a relay node. It protects your traffic and contributes to the network's capacity by relaying encrypted packets for others (you cannot see the content — it remains encrypted).
- **Use when**: You are a home/office user and do not manage a community mesh network.

**Option B: Relay + Mesh Gateway (Domain Bridge)**
- **What it does**: In addition to relaying, your router acts as a **Domain Bridge Node**. It uses its LoRa and/or WiFi 802.11s radios to connect to a local mesh island and bridges that mesh island's traffic to the clearnet internet via your WAN.
- **Use when**: You are operating a gateway for a community mesh island. You will need to provide the Mesh Island ID.

**Option C: Range Extension (Relay Optimized)**
- **What it does**: Prioritizes forwarding BGP-X traffic over originating new sessions. Useful for coverage extension in low-density areas.
- **Use when**: Deploying on a mast or solar-powered pole primarily to extend mesh coverage.

### 4.6 Step 5: BGP-X Identity

The router generates its permanent cryptographic identity:

1.  Click **Generate Key**. The router's TPM 2.0 security chip generates a new Ed25519 keypair (takes 5-10 seconds).
2.  You will see your **BGP-X NodeID** (a 32-byte hex string). This is your router's permanent identity in the BGP-X network.
3.  **Record this NodeID** if you want to track your node's reputation or manage it via the command-line interface later.

### 4.7 Step 6: LoRa and WiFi Mesh Radio Setup (Optional)

If you selected **Relay + Mesh Gateway** or want to participate in a mesh island:

**LoRa Configuration**:
1.  Select your **Region** from the dropdown. This sets the correct LoRa frequency (e.g., EU868, US915) to comply with local regulations.
2.  Enter a **Mesh Island ID**. This is a unique string identifying your local mesh network (e.g., `seattle-downtown-east`).
    - If you are joining an existing community mesh, ask your mesh administrator for the Island ID.
    - If you are creating a new mesh, create a descriptive, geographically specific ID to minimize collision probability.

**WiFi 802.11s Mesh Configuration**:
- Your router's WiFi radios can also operate as a WiFi Mesh backhaul (802.11s). The wizard will offer to create or join a WiFi mesh network. This is separate from your client WiFi SSID and is used for node-to-node links.
- Enable this if you have other BGP-X nodes nearby with WiFi mesh capability.

**If unsure, skip this step.** You can enable and configure mesh settings later via the web interface (**BGP-X → Mesh / LoRa**).

### 4.8 Step 7: Jurisdiction Declaration (Optional)

BGP-X supports **Geographic Plausibility Scoring** as an optional reputation signal.

- **Declare Jurisdiction**: If you want other nodes to verify that your node is physically located where you claim, select your country/region from the list.
- **Do Not Declare**: If you do not declare a jurisdiction, geographic plausibility scoring does not apply to your node.

**Note**: Satellite-connected nodes are automatically exempt from geo plausibility scoring because satellite terminal IPs may be assigned from distant ground stations.

### 4.9 Step 8: Complete

Click **Finish Setup**. The router applies your configuration and reboots (takes approximately 60 seconds).

After reboot:
- Reconnect your WiFi devices to your new WiFi network.
- Visit any website to confirm internet is working. All traffic is now protected by BGP-X.
- The default routing policy sends **all traffic** through the overlay. You can change this in the web interface.

---

## 5. Verifying BGP-X is Working

### 5.1 Via Web Interface

Navigate to `http://192.168.1.1` and log in with your admin password.

On the **BGP-X Dashboard**, verify:
- **Status**: `Running`
- **Active Sessions**: A number greater than 0 indicates active relay connections.
- **Connected Peers**: The number of other BGP-X nodes your router has sessions with.
- **Path Quality**: Should show a score (e.g., "Healthy" or a percentage).

### 5.2 Via Command Line (Advanced)

SSH into the router:
```bash
ssh root@192.168.1.1
bgpx-cli node stats
```
Look for `sessions_active` and `peers_connected` fields.

### 5.3 Verify Cross-Domain Connectivity (If Mesh Gateway)

If you configured your router as a Mesh Gateway:
- Check that **LoRa** or **WiFi Mesh** interfaces are listed as "Up" in the web interface.
- The router should be visible to other nodes in the mesh island.
- Clearnet users should be able to discover and connect to your mesh island via your router's domain bridge capability.

---

## 6. Connecting Devices

### 6.1 Standard WiFi and Wired Devices

Connect as you would to any standard router. Select your WiFi SSID and enter the password. All traffic from these devices is automatically routed through the BGP-X overlay (unless a bypass rule matches).

### 6.2 BGP-X Browser

Install the **BGP-X Browser** on any device on your LAN. It will auto-connect to the BGP-X Router v1's daemon. You can then:

- Browse standard clearnet websites via the overlay.
- Access **.bgpx** native services (e.g., `community-news.bgpx`).
- View real-time path indicators showing which routing domains your traffic traverses (clearnet, overlay, mesh).
- Experience latency-adaptive rendering for high-latency paths (e.g., when accessing mesh island services over LoRa).

### 6.3 Devices That Need Bypass

Some applications perform better without BGP-X overlay latency (online gaming, video calls, local streaming). You can configure bypass rules in the web interface:

**Navigate to: BGP-X → Routing Policy → Add Rule**

- **Match**: Device IP/MAC, Destination IP/Domain, or Protocol/Port.
- **Action**: `Standard` (bypass overlay, route directly to WAN).

**Example**: Create a rule for your gaming console's IP address with Action set to `Standard`.

---

## 7. LED Status Indicators

| LED State | Meaning |
|---|---|
| Solid Red (boot) | System initializing / Bootloader active. |
| Blinking Amber | Firmware loading / BGP-X daemon starting. |
| Solid Green | Normal operation. BGP-X overlay running with active sessions. |
| Slow Blink Green | BGP-X running but low session count (normal for new nodes). |
| Fast Blink Amber | BGP-X error (check web interface logs). |
| Solid Blue | LoRa radio active and receiving beacons. |
| Alternating Blue/Green | LoRa radio active and transmitting/receiving traffic. |

---

## 8. Changing Settings

All settings are managed via the web interface at `http://192.168.1.1`.

**Key sections**:

- **Network → WAN**: ISP settings (DHCP, PPPoE, Static IP, USB Satellite).
- **Network → WiFi**: Manage your client WiFi networks.
- **BGP-X → Overview**: Status, statistics, path quality graphs.
- **BGP-X → Routing Policy**: Define which traffic uses the overlay.
- **BGP-X → Mesh / LoRa**: Mesh Island ID, LoRa frequency, WiFi 802.11s settings.
- **BGP-X → Pools**: Manage trusted relay pools for path construction.
- **BGP-X → Advanced**: Enable Cover Traffic, Pluggable Transport, session timeouts.

**Advanced Configuration**:
- **N-hop unlimited**: BGP-X imposes no protocol-level maximum on path length. While the default is optimized for latency, advanced users can specify longer paths for higher anonymity.
- **Domain Segments**: In Routing Policy, you can specify cross-domain paths that traverse multiple routing domains (e.g., `clearnet → bridge → mesh:island-id`).

---

## 9. Troubleshooting

### 9.1 Internet Not Working After Setup

1.  **Check WAN Status**: Navigate to **Network → WAN**. Ensure an IP address is assigned.
2.  **Ping from Router**: SSH in and run `ping 8.8.8.8`. If this fails, the WAN link is down. Check the physical cable and ensure your ISP modem is online.
3.  **Satellite Specifics**: If using a USB satellite modem, ensure it is powered on and has a clear view of the sky.

### 9.2 BGP-X Not Routing

1.  Open **BGP-X → Overview** in the web interface.
2.  If **Status** is "Stopped", click **Start**.
3.  If **Sessions** remains 0 after 5 minutes:
    - Check if **UDP port 7474** is blocked by your ISP.
    - If blocked, enable **Pluggable Transport** in **BGP-X → Advanced** to obfuscate traffic.

### 9.3 Cannot Connect to Mesh Island

1.  Verify **Mesh Island ID** is correct.
2.  Ensure your **LoRa antenna** is connected and properly oriented.
3.  Check that other nodes in the island are operational.
4.  Verify the **Region** setting matches the frequency used by the island.

### 9.4 WiFi Speed Seems Slower

BGP-X routing adds latency (typically 50-200ms on clearnet paths). Throughput should not be significantly impacted. If you notice severe bandwidth degradation:

1.  Check **Path Quality** in the web interface. Poor quality paths may need rebuilding.
2.  Test with a bypass rule for the speed test server to compare direct vs. overlay performance.
3.  Note that LoRa mesh connections have limited bandwidth (0.3-50 Kbps). This is expected physics, not a configuration error.

### 9.5 Forgot Admin Password

**Reset to Factory Defaults**:
1.  Locate the **Reset** button (small pinhole on the back/bottom).
2.  Insert a paperclip and hold for **10 seconds**.
3.  The router will reboot with default settings.
4.  **WARNING**: This clears all configuration (WiFi, WAN, BGP-X settings). Your BGP-X NodeID (stored in TPM) is **preserved**.

### 9.6 ISP Blocking or Throttling BGP-X Traffic

If your ISP blocks UDP port 7474 or throttles BGP-X traffic patterns:

1.  Enable **Pluggable Transport** in **BGP-X → Advanced**.
2.  Select the **Built-in Obfuscation** profile (obfs4-style) or configure an external PT proxy.
3.  This makes BGP-X traffic appear as generic obfuscated data, bypassing simple filters.

---

## 10. Where to Go Next

- **BGP-X Browser User Guide**: Learn how to use `.bgpx` addresses and latency-adaptive browsing.
- **Mesh Island Setup Guide**: If you are building a community mesh, see `mesh_island_setup.md`.
- **Domain Bridge Operator Guide**: Detailed documentation for operating as a domain bridge node.
- **Advanced Routing Policy**: Learn to create rules for specific applications, devices, or destinations.
- **SDK Integration Guide**: For developers building BGP-X native applications.

For full system architecture, hardware details, and protocol specifications, refer to `ARCHITECTURE.md` and `hardware/hardware_spec.md`.
