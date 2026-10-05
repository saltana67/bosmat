# Raymarine AIS700 — USB Protocol Reference

## 1. Physical Interface

| Parameter | Value |
|-----------|-------|
| Connector | USB Micro-B (on unit) |
| Interface | Virtual COM port (USB-serial bridge) |
| Protocol | NMEA 0183 HS (IEC 61162-1) compliant, bi-directional |
| Baud rate | 38,400 baud |
| Power | USB port supplies power to unit during configuration (unplug Power/NMEA 0183 cable) |

> **Note:** Do NOT connect the AIS700 to a PC using NMEA 0183 and USB simultaneously.

**Sources:**
- [Raymarine AIS700 Installation Manual (Rev 7), §4.2](https://docs.raymarine.com/87326/en-US/latest/87326%20(Rev%207)%20(en-US).pdf)
- [Raymarine Support: PRO AIS Laptop Connection](https://support.raymarine.com/s/article/Pro-AIS-Laptop-Connection?language=en_US)
- [USP Marine — AIS700 specifications](https://uspmarine.com/products/raymarine-ais700-class-b-ais-transceiver-with-antenna-splitter-ray-e70476)
- [Marine Superstore — AIS700 specifications](https://www.marinesuperstore.com/marine-communication/ais/raymarine-ais700-class-b-ais-transceiver-with-antenna-splitter)

---

## 2. Protocol Overview

The USB link carries a **serial stream of NMEA 0183-style sentences** (standard `$`/`!` framing with `*` checksum). Mixed into the standard AIS NMEA sentences are **Raymarine proprietary sentences** using the `$PSMT` and `$U0` prefixes.

The protocol is **not officially documented** by Raymarine. All information below is compiled from:
- Official manual (limited)
- Forum user observations and captures
- The [Farfelu "Definitive Guide"](https://farfelu.life/mmsi-ais-reset-the-definitive-guide/)

---

## 3. Standard NMEA Sentences (USB stream)

| Sentence | Direction | Content |
|----------|-----------|---------|
| `!AIVDM,x,y,z,A,...` | AIS → PC | All received AIS targets (position, static, safety, etc.) |
| `$AIVDO,...` | AIS → PC | Own ship AIS data |
| `$GPGGA` / `$GPRMC` / `$GPGLL` | AIS → PC | Internal GNSS position (if enabled) |

These are standard and well-documented in NMEA 0183.

---

## 4. Proprietary `$PSMT` Sentences

### 4.1 Frame Structure

`$PSMT,F1,F2,F3,F4,command-string*NMEA-checksum`

| Field | Observed Values | Inferred Meaning |
|-------|----------------|-----------------|
| F1 | `0` | Unknown (source? protocol version?) |
| F2 | `0` or `3` | Unknown (command class?) |
| F3 | `0x2C75B2FA` or `0` | Device/protocol identifier (4-byte hex) |
| F4 | `1` | Unknown (sequence? sub-address?) |
| F5 | Command string | The actual operation |

**Escaping:** `^2C` within a command string represents an escaped comma (0x2C = `,`).

### 4.2 Known Commands

| Command | Full Sentence | Function |
|---------|--------------|----------|
| `nvdeli "mmsi"` | `$PSMT,0,3,0x2C75B2FA,1,nvdeli "mmsi",52*45` | **Delete** MMSI from NVRAM |
| `nvseti "mmsi"^2C MMSI^2C 0^2C 1^2C 3` | `$PSMT,0,3,0x2C75B2FA,1,nvseti "mmsi"^2C MMSINOXXX^2C 0^2C 1^2C 3,XXX*7A` | **Write** MMSI to NVRAM |
| `delay 100` | `$PSMT,0,3,0x2C75B2FA,1,delay 100,53*06` | 100 ms pause |
| `bootcmd 0` | `$PSMT,0,3,0x2C75B2FA,1,bootcmd 0,56*0B` | **Reboot** the unit |
| `nmeabaud 38400^2C4800,48` | `$PSMT,0,0,0,1,nmeabaud 38400^2C4800,48*3E` | Set NMEA 0183 Hi/Lo baud rates |

**Naming convention:** `nvdeli` / `nvseti` = **NVRAM** **delete/set** **integer**. The `nvseti` format appears to be:

`nvseti key^2C value^2C flags^2C type^2C length`

### 4.3 MMSI Reset Sequence (confirmed working on AIS700)

~~~
$PSMT,0,3,0x2C75B2FA,1,nvdeli "mmsi",52*45
$PSMT,0,3,0x2C75B2FA,1,delay 100,53*06
$PSMT,0,3,0x2C75B2FA,1,bootcmd 0,56*0B
~~~

After reboot, the unit is in a "first-time configuration" state and the MMSI field is unlocked in ProAIS2.

**Sources:**
- [Farfelu: MMSI AIS Reset — The Definitive Guide](https://farfelu.life/mmsi-ais-reset-the-definitive-guide/)
- [Cruisers Forum: DIY MMSI (re)programming, Page 10 — 2024 confirmation on AIS700](https://www.cruisersforum.com/forums/f13/diy-mmsi-re-programming-169815-10.html)
- [Cruisers Forum: DIY MMSI (re)programming, Page 11 — Farfelu guide referenced](https://www.cruisersforum.com/forums/f13/diy-mmsi-re-programming-169815-11.html)

---

## 5. Legacy `$PSRT` Protocol (AIS650 / AIS500 / SRT)

An older protocol that also works on some units:

| Command | Full Sentence | Function |
|---------|--------------|----------|
| Authorization | `$PSRT,012,,,(--QuaRk--)*4B` | Authenticate (password = `--QuaRk--`) |
| Reset Data Profile | `$PSRT,RDP*6F` | **Full factory reset** (clears all data including MMSI) |
| Set MMSI | `$PSRT,010,,,#########*09` | Write MMSI directly (9-digit) |

**Alternative MMSI reset (2-line sequence):**

~~~
$PSRT,012,,,(--QuaRk--)*4B
$PSRT,RDP*6F
~~~

> **Note:** The `$PSRT` protocol works on the AIS650. Its compatibility with the AIS700 is **unconfirmed** — the `$PSMT` protocol is what the AIS700 uses natively.

**Sources:**
- [Cruisers Forum: Tool available to reset AIS650 MMSI, Page 6](https://www.cruisersforum.com/forums/f121/tool-available-to-reset-ais650-mmsi-255624-6.html)
- [Farfelu: MMSI AIS Reset — The Definitive Guide (Option #2)](https://farfelu.life/mmsi-ais-reset-the-definitive-guide/)

---

## 6. `$U0` Query/Response Sentences

| Sentence | Direction | Function |
|----------|-----------|----------|
| `$U0AIQ,VSD*51` | PC → AIS | Query vessel static data |
| `$U0VSD,36,,,,,,,,*0D` | AIS → PC | VSD response (empty fields = unconfigured) |
| `$U0SSD,callsign,name,type,LoA,beam,port_off,bow_off,AI*CS` | PC → AIS (write) / AIS → PC (read) | Own ship data |

**Source:** [Farfelu: MMSI AIS Reset — The Definitive Guide](https://farfelu.life/mmsi-ais-reset-the-definitive-guide/)

---

## 7. What ProAIS2 Does Over USB

ProAIS2 (v1.24.02, by Digital Yacht) is the official configuration software. It speaks the protocol above. All functions go **exclusively over USB**:

| Function | Method |
|----------|--------|
| Read/write static data (name, call sign, type, dimensions) | `$U0VSD` / `$U0SSD` or `nvseti` |
| Initial MMSI write | `nvseti "mmsi"` |
| **Erase MMSI** (dealer version only) | `nvdeli "mmsi"` + `delay` + `bootcmd` |
| Set NMEA 0183 baud rates | `nmeabaud` |
| Diagnostics (VSWR, power, firmware, TX/RX stats, alarms) | Continuous `$PSMT` diagnostic stream |
| Silent mode toggle | Undocumented command (via Diagnostics tab) |
| Firmware update | Undocumented protocol |
| Raw serial commands | User can type any sentence in the Serial Data tab |

### Standard vs. Dealer Version

The **only** functional difference is the MMSI erase capability. The dealer-portal version of ProAIS2 has the `nvdeli` command enabled; the public version does not (the MMSI field is greyed out after first write).

**Sources:**
- [Raymarine Support: PRO AIS Laptop Connection](https://support.raymarine.com/s/article/Pro-AIS-Laptop-Connection?language=en_US)
- [Raymarine Forum: Name and Vessel Type Change in AIS700](https://forum.raymarine.com/showthread.php?tid=8328)
- [Raymarine Forum: AIS700 PROAIS2 connection issues](https://forum.raymarine.com/showthread.php?tid=8418)
- [Digital Skipper.se: Program Raymarine AIS700 with PROAIS2](https://digital-skipper.se/en/blogg/mmsi-programmering-pa-raymarine-ais700)

---

## 8. NMEA 2000 Interface (separate from USB)

The N2K/SeaTalkng backbone is a **completely separate** data path. The AIS700 uses it only for:

### Transmitted (AIS700 → N2K network)

| PGN | Description |
|-----|-------------|
| 129038 | AIS Class A Position Report |
| 129039 | AIS Class B Position Report |
| 129040 | AIS Class B Extended Position Report |
| 129041 | AIS Aids to Navigation |
| 129793 | AIS UTC and Date Report |
| 129794 | AIS Class A Static & Voyage Data |
| 129798 | AIS SAR Aircraft Position |
| 129801 | AIS Addressed Safety Related Message |
| 129802 | AIS Safety Related Broadcast Message |
| 129809 | AIS Class B "CS" Static, Part A |
| 129810 | AIS Class B "CS" Static, Part B |

### Received (N2K network → AIS700)

| PGN | Purpose |
|-----|---------|
| 129025 | Position, Rapid Update |
| 129026 | COG & SOG, Rapid Update |
| 129029 | GNSS Position Data |

> The AIS700 has a **hard-coded PGN filter**. It will ignore vendor-specific PGNs or any PGN not listed above. There is no way to extend the N2K interface.

**Source:** [Raymarine AIS700 Installation Manual (Rev 7), Appendix C](https://docs.raymarine.com/87326/en-US/latest/87326%20(Rev%207)%20(en-US).pdf)

---

## 9. What Is NOT Documented

| Item | Status |
|------|--------|
| Full `$PSMT` frame grammar (meaning of F1–F4) | Unknown |
| Complete list of all supported commands | Only 5 observed: `nvdeli`, `nvseti`, `delay`, `bootcmd`, `nmeabaud` |
| `nvseti` field semantics for keys other than "mmsi" | Inferred, untested publicly |
| Diagnostic stream sentence format (VSWR, TX/RX counters, etc.) | Not decoded |
| Silent mode toggle command | Not published |
| Firmware update protocol | Not published |
| Response/acknowledgement format for commands | Unknown (appears to be fire-and-forget) |
| The `48` parameter in `nmeabaud 38400^2C4800,48` | Unknown |
| Whether `nvseti` can write other NVRAM keys (vessel name, etc.) directly | Untested |

---

## 10. Practical: Building a USB Reader (e.g., ESP32)

### Requirements

- USB CDC interface (ESP32-S3 native, or CP2102/CH340 on classic ESP32)
- Connect to AIS700 Micro-B USB port
- Read serial stream at 38,400 baud
- Parse NMEA 0183 sentences (`$`/`!` framing, `*` checksum)

### Read (passive)

Simply listen to the stream. You will receive:
- `!AIVDM` sentences (AIS targets)
- `$AIVDO` (own ship)
- `$PSMT` diagnostic sentences (periodic)
- GNSS sentences (if enabled)

### Query (active)

Send:

~~~
$U0AIQ,VSD*51
~~~

Receive:

~~~
$U0VSD,mmsi,name,callsign,type,LoA,beam,port_off,bow_off*CS
~~~

### MMSI Reset (if needed)

Send the 3-line sequence from §4.3, wait for reboot, then use ProAIS2 or `nvseti` to write the new MMSI.

---

## 11. Source Index

### Official

| Source | URL |
|--------|-----|
| AIS700 Installation Manual (Rev 7) | [docs.raymarine.com](https://docs.raymarine.com/87326/en-US/latest/87326%20(Rev%207)%20(en-US).pdf) |
| AIS700 Quick Start Guide | [raymarine.ee](https://raymarine.ee/wp-content/uploads/2026/04/AIS700-Quick-start-guide-88100-2-1.pdf) |
| Raymarine Support: PRO AIS Laptop Connection | [support.raymarine.com](https://support.raymarine.com/s/article/Pro-AIS-Laptop-Connection?language=en_US) |
| Raymarine AIS700 Product Page | [raymarine.com](https://www.raymarine.com/en-us/our-products/ais/ais-receivers-and-transceivers/ais700-class-b-transceiver) |

### Forum / Community

| Source | URL | Relevance |
|--------|-----|-----------|
| Cruisers Forum: DIY MMSI (re)programming (main thread) | [cruisersforum.com](https://www.cruisersforum.com/forums/f13/diy-mmsi-re-programming-169815.html) | PSRT protocol, dealer costs, general discussion |
| Cruisers Forum: DIY MMSI, Page 10 (2024) | [cruisersforum.com](https://www.cruisersforum.com/forums/f13/diy-mmsi-re-programming-169815-10.html) | PSMT sequence confirmed on AIS700 |
| Cruisers Forum: DIY MMSI, Page 11 | [cruisersforum.com](https://www.cruisersforum.com/forums/f13/diy-mmsi-re-programming-169815-11.html) | Farfelu guide reference, eBay tool mention |
| Cruisers Forum: Tool to reset AIS650 MMSI, Page 6 | [cruisersforum.com](https://www.cruisersforum.com/forums/f121/tool-available-to-reset-ais650-mmsi-255624-6.html) | PSRT protocol details, step-by-step |
| Cruisers Forum: MMSI Setup on Raymarine AIS700 | [cruisersforum.com](https://www.cruisersforum.com/forums/f13/mmsi-setup-on-raymarine-asi700-245245.html) | ProAIS2 usage, dealer vs. user |
| Raymarine Forum: AIS700 PROAIS2 connection issues | [forum.raymarine.com](https://forum.raymarine.com/showthread.php?tid=8418) | FCC/dealer restrictions |
| Raymarine Forum: Name and Vessel Type Change in AIS700 | [forum.raymarine.com](https://forum.raymarine.com/showthread.php?tid=8328) | Confirmation that static data is user-changeable |

### Blog / Guide

| Source | URL | Relevance |
|--------|-----|-----------|
| Farfelu: MMSI AIS Reset — The Definitive Guide | [farfelu.life](https://farfelu.life/mmsi-ais-reset-the-definitive-guide/) | Complete PSMT/PSRT commands, step-by-step, cross-brand |
| Digital Skipper.se: Program Raymarine AIS700 with PROAIS2 | [digital-skipper.se](https://digital-skipper.se/en/blogg/mmsi-programmering-pa-raymarine-ais700) | ProAIS2 walkthrough, USB connection details |

### Retailer Specs (USB protocol confirmation)

| Source | URL |
|--------|-----|
| USP Marine | [uspmarine.com](https://uspmarine.com/products/raymarine-ais700-class-b-ais-transceiver-with-antenna-splitter-ray-e70476) |
| Marine Superstore | [marinesuperstore.com](https://www.marinesuperstore.com/marine-communication/ais/raymarine-ais700-class-b-ais-transceiver-with-antenna-splitter) |

---

## 12. Disclaimer

The `$PSMT` and `$U0` protocol details are **not officially documented by Raymarine**. All information in sections 4–6 is compiled from community observations and may be incomplete or contain errors. Use at your own risk. The MMSI reset procedure may violate local regulations (particularly in the USA, 47 CFR 80.231) and Raymarine's terms of service.
