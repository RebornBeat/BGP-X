# BGP-X Manufacturing and Factory Test

**Version**: 0.1.0-draft
**License**: CERN-OHL-S v2 (Hardware), MIT (Firmware/Software), CC BY 4.0 (Documentation)

---

## 1. Overview

This document covers manufacturing considerations, factory test procedures, and production quality assurance for all BGP-X hardware tiers.

**Covered Hardware**:
- **Tier 1**: BGP-X Router v1, BGP-X Node v1, BGP-X Gateway v1
- **Tier 2**: BGP-X Client Node (LILYGO T3S3, T-Beam, T-Echo, Heltec V3, RAK WisBlock)
- **Tier 3**: BGP-X Adapter/Dongle (ESP32-S3 / nRF52840 based)

All BGP-X hardware reference designs are released under **CERN-OHL-S v2** (Strongly Reciprocal Open Hardware License). Community manufacturers may produce compatible hardware using these specifications, provided they comply with the capability requirements and license terms.

---

## 2. Design Philosophy

### 2.1 Design for Manufacturing (DFM)

BGP-X hardware is designed for cost-effective manufacturing at quantities from 10 units (prototype) to 10,000+ units (production run).

**Component Selection Philosophy**:
- **No single-source components**: All ICs have ≥2 qualified suppliers.
- **Standard packages**: Prefer QFN, SOIC, 0402/0603 passives over exotic packages.
- **RoHS compliant**: All components RoHS 2011/65/EU compliant.
- **REACH compliant**: No SVHC substances above threshold.
- **Long-term availability**: Select components in mass production with multi-year lifecycles.

### 2.2 Design for Adaptability

BGP-X hardware is designed for adaptability at the manufacturing level:

**Modular PCB Architecture** (Tier 1: Router v1, Node v1, Gateway v1):
The core compute module (SoC + RAM + flash + TPM) can be substituted with a compatible module without redesigning the carrier PCB. This enables:
- Sourcing alternative ARM SoC modules meeting capability requirements.
- Upgrading to custom BGP-X ASICs in future revisions.
- No firmware rewrite — only HAL adaptation required.

**Compute Module Compatibility Specification**:
The core compute module must expose:
- 2× SPI buses (for SX1262 LoRa and TPM 2.0)
- 1× I2C bus (for power management IC)
- 2× USB 3.0 PHY
- 4× SGMII or RGMII Ethernet PHY connections
- PCIe 2.0 × 1 (for WiFi module)
- GPIO for reset, status LEDs, hardware watchdog
- 3.3V and 1.8V I/O

Any compute module meeting this specification can replace the reference MT7981B module without board revision.

**Radio Module Substitution**:
The SX1262 LoRa module connects via 4-wire SPI + 2 GPIO (busy, reset). Any LoRa module with identical pinout (e.g., LLCC68, SX1268) works with the same footprint and the parameterized `bgpx-sx1262` driver.

---

## 3. General Assembly Process

### 3.1 SMT Process

```
Solder paste → Screen print → Component placement (pick and place) → Reflow oven → Inspection
```

- **Solder paste**: Sn63Pb37 (standard) or SAC305 (lead-free for RoHS).
- **Reflow profile**: Per IPC-7711/7721; consult paste datasheet.
- **Inspection**:
  - AOI (Automated Optical Inspection) after reflow.
  - ICT (In-Circuit Test) for production runs >100 units.
  - X-ray inspection for BGA packages (if any).

### 3.2 Manual Assembly

After SMT:
- Through-hole connectors (RJ45, USB-C, DC jack): wave solder or hand solder.
- SMA/N-type antenna connectors: hand solder with flux.
- Press-fit connectors (if used): hydraulic press.

### 3.3 Conformal Coating (Outdoor Variants)

For Node v1 and outdoor Router v1 variants:
- Apply conformal coating to PCB per IPC-CC-830 Class B.
- Ensure connectors and test points are masked or use selective coating.
- Thickness: 25-75 µm typical.

---

## 4. BGP-X Router v1 Manufacturing

### 4.1 PCB Specifications

| Parameter | Value |
|---|---|
| Layers | 4 (signal, ground, power, signal) |
| Material | FR-4, Tg≥150°C |
| Copper weight | L1/L4: 1 oz, L2/L3: 1 oz (power planes may use 2 oz) |
| Min trace/space | 4/4 mil |
| Min via drill | 0.3mm |
| Surface finish | HASL-LF or ENIG (preferred) |
| Solder mask | Green or black |
| Silkscreen | White both sides |
| Board thickness | 1.6 mm |

### 4.2 Component Sourcing

Critical components with long lead times:
- **SoC module** (MT7981B-based): 12-16 week lead time; maintain 3-month safety stock.
- **SX1262 LoRa module**: 8-12 week lead time.
- **TPM 2.0 IC** (SLB9670/SLB9672): 4-8 week lead time.
- **eMMC**: 4-8 week lead time.

**Alternative sourcing for LoRa (SX1262)**:
- **Primary**: Semtech SX1262MB2CAS (SPI module)
- **Alternative 1**: LLCC68 (pin-compatible, lower max power)
- **Alternative 2**: RAK4200 LoRa module (castellated; requires PCB footprint change)
- **Alternative 3**: Custom SX1262 PCB sub-assembly for high-volume production

### 4.3 Assembly Process

1. SMD component placement (automated pick-and-place).
2. Reflow solder (lead-free SAC305).
3. Through-hole component insertion (antenna connectors, USB-A ports, RJ45 jacks).
4. Wave solder or selective solder for through-hole components.
5. Visual inspection (AOI).
6. Carrier board assembly (attach core PCB to carrier; install into enclosure).
7. Factory test (see Section 4.5).
8. Firmware flash (bgpx-router-firmware via USB/UART).
9. TPM enrollment (see Section 4.6).
10. Final QC inspection.
11. Packaging.

### 4.4 Firmware Flash Procedure

Factory flash uses a dedicated flash jig with UART access to the SoC:

```bash
# Connect flash jig to UART header
# Boot SoC into UART download mode (GPIO strapping)

# Flash U-Boot bootloader
bgpx-flash --target router-v1 --image bootloader.bin --addr 0x0

# Flash OpenWrt kernel and root filesystem
bgpx-flash --target router-v1 --image bgpx-router-firmware-v0.1.0-router-v1-sysupgrade.bin

# Verify flash
bgpx-flash --verify

# Boot and verify daemon starts
screen /dev/ttyUSB0 115200
# Expected: "BGP-X Router v1 booting..."
# Expected: "bgpx-node: daemon started"
# Expected: "bgpx-node: DHT bootstrap in progress"
```

Total flash time: ~3 minutes per unit.

### 4.5 Factory Test Procedure — BGP-X Router v1

The factory test binary (`bgpx-factory-test`) runs automatically after firmware flash. All tests must PASS before unit is approved for shipping.

```
FACTORY TEST: BGP-X Router v1
Unit serial: [auto-generated]
Firmware: bgpx-router-firmware-v0.1.0
Test date: [timestamp]

[PASS/FAIL] TPM 2.0: tpm2_startup --clear succeeds
[PASS/FAIL] TPM HWRNG: 32 bytes entropy generated in <500ms
[PASS/FAIL] TPM key generation: Ed25519 test keypair generated in TPM NV
[PASS/FAIL] TPM signing: test message signed and verified in <100ms

[PASS/FAIL] LoRa SX1262: SPI version register read; expected 0x22
[PASS/FAIL] LoRa TX: transmit test packet at +14 dBm; duty cycle accounting incremented
[PASS/FAIL] LoRa RX: receive test packet from jig at -80 to -40 dBm RSSI
[PASS/FAIL] LoRa loopback: unit-to-jig round-trip within 500ms

[PASS/FAIL] WiFi 2.4 GHz: 802.11s mesh association with test AP; RSSI > -70 dBm
[PASS/FAIL] WiFi 5 GHz: 802.11s mesh association with test AP; RSSI > -70 dBm
[PASS/FAIL] WiFi 802.11s: mesh peering with test peer node succeeds

[PASS/FAIL] WAN GbE: link up at 1000BASE-T; 10,000 packet loopback; 0 errors
[PASS/FAIL] LAN GbE Port 1: link up; 1000 packet loopback
[PASS/FAIL] LAN GbE Port 2: link up; 1000 packet loopback
[PASS/FAIL] LAN GbE Port 3: link up; 1000 packet loopback

[PASS/FAIL] USB 3.0 Port 1: enumerate test device; SuperSpeed negotiated
[PASS/FAIL] USB 3.0 Port 2: enumerate test device; SuperSpeed negotiated

[PASS/FAIL] eMMC: write 64 MB test pattern; read back; 0 bit errors
[PASS/FAIL] eMMC: SMART check; no reallocated sectors

[PASS/FAIL] Secure boot: TPM PCR[0] matches expected boot measurement for this firmware
[PASS/FAIL] Encrypted storage: eMMC key sealed in TPM; mount/unmount succeeds

[PASS/FAIL] BGP-X daemon: starts without errors
[PASS/FAIL] BGP-X daemon: TUN interface (bgpx0) created
[PASS/FAIL] BGP-X daemon: SDK socket accessible at /var/run/bgpx/sdk.sock

[PASS/FAIL] DOMAIN BRIDGE TEST:
  - Configure clearnet domain (loopback) + WiFi mesh domain
  - Construct test cross-domain path via DOMAIN_BRIDGE hop
  - Verify DOMAIN_BRIDGE hop dispatched to WiFi mesh transport
  - Verify cross-domain path_id table entry created
  - Verify return traffic routed via cross-domain path_id lookup
  - ALL domain bridge sub-tests must pass

[PASS/FAIL] N-HOP TEST:
  - Construct 15-hop onion in local test mode
  - Verify all layers decrypt correctly in sequence
  - Verify no hop count limit enforced

RESULT: [PASS / FAIL]
Failed tests: [list if any]
Unit approved: [YES / NO]
```

### 4.6 TPM Enrollment Procedure

After factory test PASS, each unit receives its permanent TPM enrollment:

```bash
# Generate BGP-X node keypair in TPM (permanent; linked to this unit's serial)
bgpx-keygen --tpm --output-nv-index 0x01000001

# Record public key (NodeID) in manufacturing database
NODE_ID=$(bgpx-keytool show-public --nv-index 0x01000001)
bgpx-mfg-db insert --serial $SERIAL --node-id $NODE_ID

# Seal storage key to expected boot PCR values
tpm2_nvdefine -C o -s 32 -a "ownerread|ownerwrite" 0x01500001
tpm2_nvwrite -C o -i /dev/urandom 0x01500001  # storage encryption key

# Record enrollment
echo "Unit $SERIAL enrolled at $(date); NodeID: $NODE_ID" >> /var/log/bgpx-mfg.log
```

The NodeID (BLAKE3 of the TPM-generated Ed25519 public key) is recorded in the manufacturing database. This is used for warranty, support, and tracking.

---

## 5. BGP-X Node v1 Manufacturing

### 5.1 PCB Specifications

Smaller than Router v1; solar MPPT and battery management circuitry on main board.

| Parameter | Value |
|---|---|
| Layers | 4 |
| Copper weight | 2 oz (power; heavier for solar current handling) |
| Dimensions | ~100 × 60 mm (compact outdoor form factor) |
| IP67 sealing | Conformal coating on PCB; IP67 enclosure assembly |
| Temperature | -40°C to +85°C rated components throughout |

### 5.2 Component Sourcing

Same critical components as Router v1 (SoC, LoRa, TPM). Additional:
- **MPPT charge controller**: CN3791 or equivalent (6-28V input).
- **LiFePO4 battery cell**: 3Ah minimum for overnight operation.

### 5.3 Factory Test Procedure — BGP-X Node v1

Includes all core tests from Router v1 (TPM, LoRa, WiFi, USB, eMMC, daemon) plus Node v1-specific tests:

```
[PASS/FAIL] MPPT circuit: apply 6V 200mA to solar input; verify charging at expected current
[PASS/FAIL] Battery charge: charge 3.7V LiPo from 3.5V to 3.7V; verify charge current correct
[PASS/FAIL] Battery voltage sense: verify ADC reads within 2% of reference voltage
[PASS/FAIL] Power management daemon: verify bgpx-power reports correct battery level
[PASS/FAIL] LoRa TX at reduced power (10 dBm): power management throttling test
[PASS/FAIL] WiFi disable at 15% battery: simulate low battery; verify WiFi auto-disables
[PASS/FAIL] Graceful shutdown at 5%: simulate critical battery; verify NODE_WITHDRAW published before shutdown
```

### 5.4 IP67 Assembly Test

After enclosure assembly:

```
[PASS/FAIL] IP67 pressure test: seal enclosure; apply 1 PSI positive pressure; hold 30s; verify no leak
[PASS/FAIL] Cable gland seal: verify Ethernet + power cable gland sealed with zero air gap
[PASS/FAIL] Antenna port seal: verify SMA and RP-SMA connectors sealed with O-ring gaskets
```

---

## 6. BGP-X Gateway v1 Manufacturing

### 6.1 PCB Specifications

| Parameter | Value |
|---|---|
| Layers | 6-8 (high-density routing for multiple high-speed interfaces) |
| Copper weight | 1 oz signal, 2 oz power/ground planes |
| Dimensions | Fit 1U rack (19" width, <250mm depth) |
| Thermal | Thermal vias under high-power components (SoC, PHYs) |

### 6.2 Factory Test Procedure — BGP-X Gateway v1

Includes all core tests plus Gateway v1-specific tests:

```
[PASS/FAIL] TPM 2.0: full self-test
[PASS/FAIL] SFP+ cage: insert test SFP+ module; 10GBASE-SR link established
[PASS/FAIL] 2.5GbE WAN × 2: link up; 10,000 packet loopback each
[PASS/FAIL] GbE LAN × 4: link up; loopback each
[PASS/FAIL] USB 3.0 × 2: enumerate test device each port
[PASS/FAIL] eMMC: write/read 64 MB; 0 errors
[PASS/FAIL] Thermal: run bgpx-node at 100% CPU for 5 minutes; CPU temp < 75°C; fan responds
[PASS/FAIL] BGP-X daemon: start with max_sessions=50000 config; verify memory allocation
[PASS/FAIL] Throughput test: 10,000 simultaneous simulated sessions; forwarding latency p99 < 2ms
[PASS/FAIL] EXIT POLICY: load test exit policy; verify signature valid; verify connection permit/deny correct
[PASS/FAIL] ECH: test ECH negotiation with test destination
[PASS/FAIL] DoH: test DoH resolver with DNSSEC validation; verify ECS stripping
```

---

## 7. BGP-X Client Node Manufacturing (Tier 2)

### 7.1 Reference Hardware

Client Node firmware targets existing development boards: LILYGO T3S3, T-Beam, T-Echo, Heltec V3, RAK WisBlock.

### 7.2 Manufacturing Approach

Two paths:

1.  **Pre-built Hardware**: Source reference devices from LILYGO, Heltec, RAK. Flash `bgpx-client-firmware` using standard tools (`esptool`, `adafruit-nrfutil`).
2.  **Custom PCB**: Follow capability requirements in `client_node_spec.md`. BOM cost target <$40.

### 7.3 Factory Test Procedure — Client Node

```
[PASS/FAIL] Power: USB / battery input verified; charging circuit functional
[PASS/FAIL] LoRa: TX/RX loopback with test jig
[PASS/FAIL] WiFi: association test
[PASS/FAIL] BLE: advertisement verified
[PASS/FAIL] Firmware boot: bgpx-client-firmware starts
[PASS/FAIL] Session test: establish session with test mesh entry node
```

---

## 8. BGP-X Adapter/Dongle Manufacturing (Tier 3)

### 8.1 Reference Hardware

ESP32-S3-WROOM-1 + SX1262 module.

### 8.2 Manufacturing Approach

Low-cost USB dongle production. Reference design in `adapter_spec.md`.

### 8.3 Factory Test Procedure — Adapter

```
[PASS/FAIL] USB enumeration: device appears as CDC-ACM (/dev/ttyACMx)
[PASS/FAIL] Modem protocol: respond to CMD_GET_STATUS
[PASS/FAIL] LoRa TX: transmit test packet
[PASS/FAIL] LoRa RX: receive test packet from jig
```

---

## 9. Enclosure Manufacturing

### 9.1 Indoor ABS Enclosure (Router v1 Desktop)

- **Material**: ABS plastic.
- **Wall thickness**: 2mm minimum.
- **Boss height for PCB**: 4mm standoffs.
- **Manufacturing**: Injection molding (production) or FDM 3D print (prototype).

### 9.2 Outdoor IP67 Enclosure (Router v1 Outdoor, Node v1)

- **Material**: Polycarbonate or ABS-PC blend.
- **Gasket**: EPDM rubber.
- **Cable glands**: M12 or PG7 nylon (2 min per unit).
- **Manufacturing**: Injection molding.
- **IP rating**: Verified per IEC 60529 method.

### 9.3 Prototype Enclosures

For prototyping: standard Hammond enclosures.
- **Indoor**: Hammond 1455J (aluminum) or 1593K (ABS).
- **Outdoor**: Hammond 1554 series (ABS, IP67).

---

## 10. Regulatory Certifications (Required for Sale)

For sale as a finished product in major markets:

| Market | Certification | Applies To |
|---|---|---|
| EU | CE (RED, LVD, RoHS) | All variants |
| UK | UKCA | All variants |
| US | FCC Part 15 (unintentional) | All variants |
| US | FCC Part 15 (intentional) | LoRa, WiFi |
| Canada | ISED RSS-247 | WiFi |
| Canada | ISED RSS-210 | LoRa |
| Japan | MIC ARIB | WiFi, LoRa |

**Timeline**: CE/FCC certification typically 4-8 weeks; budget $5,000-15,000 USD per variant.

**Note**: Selling hardware with integrated radios without certification is illegal in most jurisdictions. CE/FCC mark must appear on product and packaging.

---

## 11. Open Hardware Manufacturing

Community manufacturers building BGP-X hardware from open reference designs must:

1.  **Use capability requirements** from `router_spec.md`, `node_spec.md`, `gateway_spec.md`, `client_node_spec.md`, `adapter_spec.md` — not locked chip part numbers.
2.  **Run the published factory test procedure** and achieve PASS on all tests.
3.  **Flash the official firmware** (`bgpx-router-firmware`, `bgpx-node-firmware`, etc.) — not modified versions, unless explicitly releasing as a fork under CERN-OHL-S.
4.  **Include a label** identifying the hardware as "BGP-X Router v1 compatible" or "BGP-X Node v1 compatible" — not simply "BGP-X Router v1" unless exactly following the reference design.
5. **CERN-OHL-S compliance**: publish full source files (schematics, PCB layout, BOM) for any manufactured unit.

The BGP-X project welcomes community manufacturing and will maintain a directory of verified community-manufactured compatible hardware.
