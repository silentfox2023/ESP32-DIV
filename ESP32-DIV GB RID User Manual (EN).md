# ESP32-DIV Drone Remote ID User Manual (EN)

**Firmware:** ESP32-DIV v1.9.3 · **DroneX:** v2.1.0 · **Hardware:** ESP32-DIV V2 (ESP32-S3, 16 MB flash)

This manual covers the drone Remote ID features, with emphasis on the Chinese **GB 46750-2025** standard. For the full DroneX and Drone tool guide, see [Drone_Feature.md](Drone_Feature.md).

---

## 1. Menu map

| Feature | Location | Purpose |
| --- | --- | --- |
| **RID Scanner** | `More > Drone > RID Scanner` | Fast Wi-Fi + BLE RID scan |
| **RID Spoof** | `More > Drone > RID Spoof` | Multi-protocol RID transmitter |
| **Drone Deauth** | `More > Drone > Drone Deauth` | Selective 802.11 deauth against identified drone links |
| **Drone Jelly** | `More > Drone > Drone Jelly` | 2.4 GHz carrier / sweep for drone bands |
| **DroneX** | Main menu | Continuous detection, tracking, web dashboard, simulator, Wi-Fi deauth lab |

**DroneX is a separate system.** Selecting the DroneX tile saves it as the boot system and reboots. Leave it with **Info > Exit DIV**, or hold **SELECT** for 1.5 seconds.

The `More > Drone` layer is protected by a device MAC whitelist. An unauthorized device can still open the layer, but only sees a license notice page. DroneX uses a separate ECDSA license.

---

## 2. Supported RID formats

| Format | Typical devices | Bearer |
| --- | --- | --- |
| **ASTM F3411** | Autel, overseas DJI, other compliant manufacturers | Wi-Fi vendor IE (`FA:0B:BC` / `90:3A:E6`), BLE service data `0xFFFA` |
| **GB 46750-2025** | New domestic-firmware DJI (Mavic 4 Pro, Air 3S, updated Mavic 3, etc.) | Wi-Fi vendor IE (`FA:0B:BC`), GB frame header |
| **DJI proprietary DroneID** | Older DJI models | Wi-Fi vendor IE OUI `26:37:12` |

GB 46750-2025 became mandatory on 2026-05-01. Its Wi-Fi broadcasts reuse the ASD-STAN vendor OUI but carry a different body, so older firmware could receive the frames yet show no target. v1.9.3 decodes all three formats and auto-identifies them without a mode switch.

GB frames are only broadcast after the aircraft obtains a GNSS fix, and they depend on the region and aircraft firmware. Outdoor testing is required.

---

## 3. RID Scanner (`More > Drone > RID Scanner`)

### 3.1 Operation

1. Open `More > Drone > RID Scanner`. Scanning starts automatically.
2. The scanner alternates a **6.5-second Wi-Fi phase** (channels 1-13, 500 ms each) with a **5-second BLE phase**.
3. A target turns gray after **5 seconds** without an advertisement and is removed after **30 seconds**.
4. Press **SELECT** to exit. The channel mode is reset to `AUTO / CH1`.

### 3.2 Screen fields

```text
RID SCAN
[ WIFI ] [ AUTO ] [ CH 6 ]
RID WIFI CH:6 AUTO
DRN 3  ON 2                    148 pkt
Scanning  42s

IDENTITY  MAC / RSSI                       1/1
#01 1581F8LQxxxxxxxx
MAC C4D1  IF 1  RSSI -62  CH 6
31.23045,121.47370

MGMT123 BLE45 ODID0
NAN0 BCN5 DJI0 FAIL1
MGB320 VIE88 GB3
SEL=exit
```

| Field | Meaning |
| --- | --- |
| `DRN` / `ON` | Targets in the list / currently online |
| `pkt` | RID packet count (NAN + RID beacons + BLE ODID) |
| `MGMT` | Management frames observed |
| `BLE` | BLE advertisement callbacks |
| `ODID` | BLE ODID messages decoded |
| `NAN` | Wi-Fi NAN RID broadcasts |
| `BCN` | RID beacons |
| `DJI` | DJI proprietary frames |
| `FAIL` | Suspected RID frames that failed parsing |
| `MGB` | All beacons observed |
| `VIE` | Vendor information elements |
| `GB` | GB 46750 frames decoded |

- **`GB:n`** should grow about once per second for a GB drone. A growing `FAIL` with no `GB` means the header matched but a field failed validation.
- **`IF:n`** is the interface type: 1 = Wi-Fi beacon (typical for GB), 2 = Wi-Fi NAN, 3 = BLE 4 legacy advertising, 4 = BLE 5 extended advertising.
- Each list row shows the 20-byte product serial number, the last two MAC bytes, interface, RSSI, channel, and position.

### 3.3 Channel modes and controls

| Control | Action |
| --- | --- |
| `LEFT` / `RIGHT` | Step the locked channel; either key switches AUTO to LOCK |
| `DOWN` | Toggle AUTO / LOCK |
| `UP` | Page the list |
| `SELECT` | Exit |
| Touch bar | `Ch-`, `Lock`/`Auto`, `Exit`, `Page`, `Ch+` |

- **AUTO** sweeps channels 1-13.
- **LOCK** stays on one channel (default 6) for a higher hit rate; targets on other channels are invisible while locked.

### 3.4 Low heap and Wi-Fi failure

The Wi-Fi allocation is guarded by a heap gate. If the Wi-Fi driver cannot start, or the gate refuses the allocation, the scanner continues in **BLE-only** mode. The screen shows `BLE only (WiFi init failed)` or `BLE only (LOW HEAP)` together with the free and required heap values.

### 3.5 Serial diagnostic output

When a GB frame has a matching header but fails field validation, the serial port prints the raw frame (at most one per second):

```text
[rid-dump] reason:-5 rssi:-64 ch:6 mac:C4:D1:xx:xx:xx:xx ie_len:83
FA 0B BC 0D B0 FF 20 48 FF FF FE 2A 31 35 38 31 46 ...
```

`reason` is the GB decode error code (Section 5). `reason:1` is an ASTM parse failure.

---

## 4. GB decoding reference

### 4.1 Standard fields

| Item | Content | Device usage |
| --- | --- | --- |
| 001 | Unique product identifier (20-byte serial number) | Target ID, model inference, black/white lists |
| 002 | Last 8 digits of the real-name registration number (UIN) | Operator ID, list matching |
| 004 | Drone classification (micro / light / small / medium / large) | Decoded and preserved |
| 005-007 | Control station / takeoff point coordinates and altitude | Pilot position in target details |
| 008 | Drone coordinates | Target position, map, distance and bearing |
| 009 | Track angle (0.1°) | Heading |
| 010 | Ground speed (0.1 m/s) | Horizontal speed |
| 011 | Altitude relative to takeoff point (0.5 m) | Relative altitude |
| 012 | Vertical speed (0.5 m/s) | Climb / descent rate |
| 013/014 | Geodetic altitude / barometric altitude | Absolute altitude |
| 015 | Operational status | Ground / air / emergency / RID failure |
| 020 | Unix millisecond timestamp | Time validation |

Operational status: 1 = ground, 2 = air, 3 = emergency, 4/5 = RID invalid. The device treats 4/5 as a system failure and raises an alert.

### 4.2 Frame signature

```text
Wi-Fi Beacon → Vendor IE (0xDD):
  FA 0B BC 0D [counter] FF 2x [DataLen] [flags] [21 data items ...]
                         └─ type 0xFF, version high 3 bits = 001 (0x20)
```

The decoder requires the GB type/version and strict field validation, so a normal Wi-Fi hotspot is not misidentified.

### 4.3 Decode error codes

| reason | Meaning | Diagnosis |
| --- | --- | --- |
| -1 | Parameter / length error | Incomplete frame, usually signal loss |
| -3 | Type / version mismatch | Not a GB packet or an unsupported version |
| -4 | Data length mismatch | Encapsulation difference or truncation |
| -5 | Required field missing, non-printable serial, or value out of range | Most common for a new model; save the hex dump |

If `reason:-5` repeats for a newly released model, save the full `[rid-dump]` / `[gb-dump]` output so compatibility can be added.

---

## 5. DroneX GB behavior

DroneX runs as a separate system and performs continuous BLE + Wi-Fi RID detection.

### 5.1 Entry and exit

- Select **DroneX** on the main menu. The device reboots into DroneX.
- Exit from **Info > Exit DIV**, or hold **SELECT** for 1.5 seconds. The device reboots back to ESP32-DIV.
- A missing license shows the lock screen; it returns to ESP32-DIV after 5 minutes or on a SELECT long press.

### 5.2 GB targets

- GB and legacy targets appear in the **same list**, merged by MAC. No mode switch is needed.
- DroneX keeps up to 6 live targets; the Scan tab shows 3 cards and the browser shows the full list.
- A target turns gray after **8 seconds** without an advertisement and is removed after **30 seconds**.
- The target detail page shows model, serial number, position, altitude, speed, heading, pilot/takeoff point, distance, and operator ID.
- A new target raises an audible alert; emergency, fault, and blacklisted targets have dedicated alarms.
- Black and white lists match the GB serial number (Item 001) and the last 8 digits of the real-name registration number (Item 002).
- GB targets create SD track sessions exactly like legacy targets:
  - `/dronerx/history.csv`
  - `/dronerx/tracks/<sessionId>.csv`

### 5.3 Serial output

```text
[NEW] DJI Mavic 4 Pro c4:d1:xx:xx:xx:xx via-WiFi(GB) rssi=-62
[HB] up=120 clients=0 rid=130 ble=12 wifi=128 dji=0 gb=130 sniff=1 ...
```

- `gb=` is the GB frame count; `wifi=` is the legacy ASTM count and `dji=` the legacy DJI count.
- The web status JSON exposes the same value as `"gb": <count>`.
- GB dump output uses the `[gb-dump]` tag and follows the same error codes as RID Scanner:

```text
[gb-dump] reason:-5 rssi:-70 len:83
FA 0B BC 0D 12 FF 20 48 FF FF FE ...
```

### 5.4 Web dashboard

| Setting | Default |
| --- | --- |
| Hotspot SSID | `DroneX-xxxx` |
| Hotspot password | `dronerx88` |
| URL | `http://192.168.4.1` |
| mDNS | `http://dronerx.local` when heap allows |

The root page offers a mobile UI (`/m`) and a PC UI (`/pc`). The hotspot stays on in STA mode, but the single radio is then locked to the router channel, so only Wi-Fi RID on that channel is received; BLE detection continues.

### 5.5 Wi-Fi channel strategy

| Condition | Behavior |
| --- | --- |
| STA connected | Radio locked to the router channel |
| AP client connected, no STA | Stays on the AP anchor channel (6) with grace periods |
| Idle, RID seen in the last 20 s | Anchor 900 ms + two alternate channels (channel 6 plus 1/11, depending on the anchor) for 250 ms each, then 4 s AP window |
| Idle, no RID for 20 s | Channels 1-13 with 800 ms on 1/6/11 and 300 ms on the others, then 4 s AP window |

The AP window lets phones associate without the AP hopping away. In STA mode, use the AP-only configuration when full-channel Wi-Fi RID coverage is needed.

### 5.6 DroneX serial commands

| Command | Action |
| --- | --- |
| `DRONEXKEY,<license>` | Activate DroneX |
| `HOME,<lat>,<lon>` / `HOME,CLEAR` | Set or clear the observer point |
| `LIST` | Print the target list |
| `SIM START` / `SIM STOP` / `SIM SWAP` / `SIM PAT` | Control the simulator |
| `SIM N <1-3>` | Set the simulated target count |
| `STA,<ssid>,<pass>` / `STA,DISCONNECT` / `STA,CLEAR` | Manage the router STA connection |
| `VERBOSE,1` / `VERBOSE,0` | Enable or disable detailed logs |
| `help` | Show command help |

The default observer point is `26.4875092, 106.7232399`, persisted in NVS.

---

## 6. Test procedure

1. Flash firmware v1.9.3 and open the serial monitor at 115200 bps.
2. Test **outdoors with open sky**. GB RID requires a GNSS fix on the aircraft.
3. Use a domestic-firmware drone and the DJI app. Overseas firmware or locations normally broadcast ASTM F3411 instead.
4. Power on the aircraft. GB requires a broadcast interval of 1 second or less, so a target should appear within about 1 second.
5. **RID Scanner:** watch the `GB` counter grow. `FAIL` growing without `GB` means the header matched but decoding failed.
6. **DroneX:** watch `gb=` grow and the model name appear in the target list.
7. **Closed-loop transmitter test:** start **RID Spoof** (`More > Drone > RID Spoof`), then open RID Scanner or DroneX. RID Scanner should show N targets with different MACs and growing `BCN` / `VIE` / `DJI` / `GB` counters. DroneX should show multiple drones with model names and positions.
8. If no target appears but `FAIL` or dump output grows, save the hex dump from step 5 or 6.

---

## 7. Capability boundaries

- **2.4 GHz only.** ESP32-S3 hardware cannot receive or transmit 5.8 GHz RID.
- **BLE 5 extended advertising is not decoded.** Only legacy 31-byte BLE advertising is supported.
- **GB gating.** GB broadcasts depend on a GNSS fix, the region, and the aircraft firmware.
- **Single radio.** Wi-Fi and BLE features are mutually exclusive while active. In STA mode the radio is locked to the router channel, so full-channel Wi-Fi RID requires AP-only operation.
- **DroneX capacity.** Up to 6 live targets; the Scan tab shows 3 cards.
- **RID Spoof scope.** It emits 802.11-borne RID frames; it does not emulate OcuSync or Lightbridge links.
- **No 5 GHz / no OcuSync.** Drone Deauth only affects 802.11 links.
- **Authorized use only.** Deauthentication and jamming features must be used only on equipment and networks you own or are explicitly authorized to test.

---

## 8. Troubleshooting

**No GB targets appear.**
Check that the aircraft has a GNSS fix, is outdoors, uses domestic firmware, and is within 2.4 GHz range. GB broadcasts are gated by region and firmware.

**`FAIL` grows but `GB` does not.**
The frame header matches but a mandatory field failed validation. Save the `[rid-dump]` / `[gb-dump]` hex output.

**RID Scanner shows `BLE only`.**
The Wi-Fi driver failed or the heap gate rejected the Wi-Fi allocation. Close other radio features, reboot, and retry. The screen shows the free and required heap values.

**DroneX shows no targets but the counters grow.**
Open the web UI at `http://192.168.4.1`; the on-device Scan tab shows only 3 cards and the first 6 live targets.

**The Drone menu shows a license page.**
The device MAC is not in the Drone whitelist. Drone and HaleHound share this whitelist; DroneX uses a separate ECDSA license.

**DroneX heartbeat shows `sniff=0`.**
This is not normal in STA mode; the radio should stay locked to the router channel with sniffing enabled. A persistent `sniff=0` points to a failed sniffer task or very low heap. Reboot and check the `[MEM]` and `[HB]` lines.
