# ESP32-DIV New Feature User Manual
(GB 46750-2025 National Standard Drone RID Detection + DroneX v2.0.0 Brand Upgrade + RID Multi-Protocol Transmission)

**Firmware**: ESP32-DIV v1.8.1 (from 2026-10-14; DroneX module upgraded to v2.0.0; RID transmitter added 2026-10-15)
**Hardware**: ESP32-DIV V2 (ESP32-S3, 16MB Flash)
**Affected Functions**: **Tools → RID Scanner**, **Tools → DroneX**, **Tools → RID Transmitter**

---

## 1. What This Update Fixes

Starting May 1, 2026, **GB 46750—2025 "Civil Unmanned Aerial Vehicle System Operation Identification Specification"** becomes mandatory. New DJI drones (Mavic 4 Pro, Air 3S, Mavic 3 series updated to new firmware, etc.) use a National Standard data format for WiFi broadcast that is **incompatible** with the old ASTM F3411 format. This caused old firmware to show "signals being received but targets not appearing".

This update adds National Standard decoding to both **RID Scanner** and **DroneX**. The two formats are auto-identified and don't interfere with each other:

| Broadcast Format | Applicable Devices | Before Update | After Update |
|---|---|---|---|
| DJI Proprietary DroneID (OUI 26:37:12) | Older DJI models | Supported | Supported (unchanged) |
| ASTM F3411 (OUI FA:0B:BC / 90:3A:E6) | Autel, overseas DJI, etc. | Supported | Supported (unchanged) |
| **GB 46750-2025 National Standard** | **New DJI (domestic firmware)** | **Not recognized** | **Newly added support** |

> Note: National Standard WiFi broadcasts are still encapsulated in ASD-STAN Vendor IE (`FA:0B:BC`), but the data body is completely different. The device auto-distinguishes via the National Standard frame header (type `0xFF` + high 3 version bits `001`) and strict field validation — it will never misidentify a regular WiFi hotspot.

---

## 2. Flashing Requirements

- Board: `ESP32S3 Dev Module`
- Flash Size: **16MB (128Mb)**
- Partition Scheme: **Huge APP (3MB No OTA/1MB SPIFFS)** (firmware is ~2.60MB, default partition won't fit)
- PSRAM: OPI PSRAM (match your board's actual config)
- Serial Monitor baud: **115200**

---

## 3. RID Scanner Usage (Tools → RID Scanner)

### 3.1 Operation

1. Enter **Tools → RID Scanner** from main menu — device auto-starts time-shared WiFi (channel 1~13 sweep) and BLE scanning.
2. Screen refreshes every ~0.3 seconds. Press **SEL key** to exit.
3. Drones with 5 seconds of no broadcast turn gray; auto-cleared after 30 seconds.

### 3.2 Screen Fields

```
RID WIFI  CH:6            ← Current phase (WIFI/BLE) and channel
Online:1  Total:1         ← Online count / cumulative count
MGMT:123 BLE:45 ODID:0    ← Management frames / BLE callbacks / BLE ODID counts
NAN:0 BCN:5 DJI:0 FAIL:1  ← NAN / RID beacons / DJI proprietary / parse failures
MGB:320 VIE:88 GB:3       ← All beacons / Vendor IE / 【National Standard count (new)】

C4D1  if:1  RSSI:-62      ← Last 2 bytes of MAC / Interface type / Signal strength
ID:1581F8LQxxxxxxxx       ← Product serial number (National Standard item 001)
31.23045,121.47370        ← Drone GPS coordinates
```

- **`GB:n` (New)**: Number of successfully decoded National Standard RID frames. A growing count (National Standard broadcast interval ≤1 second) confirms successful recognition.
- **`if:n` Interface type**: 1=WiFi Beacon (National Standard drones show this), 2=WiFi NAN, 3=BLE4 broadcast, 4=BLE5 extended broadcast.
- **`FAIL:n`**: Count of suspected RID frames that failed to decode (includes National Standard frames that failed field validation).

### 3.3 Serial Diagnostic Output

When National Standard frames have a matching header but fail field validation, the serial automatically dumps raw data (max 1 per second) for checking new model encapsulation:

```
[rid-dump] reason:-5 rssi:-64 ch:6 mac:C4:D1:xx:xx:xx:xx ie_len:83
FA 0B BC 0D B0 FF 20 48 FF FF FE 2A 31 35 38 31 46 ...
```

- `reason` negative = National Standard decode error code (see Section 6); `reason:1` = ASTM parse failure.
- To disable dump output: change the `RID_DEBUG_DUMP_VENDOR` macro in `RIDScanner.cpp` to `0` and recompile.

---

## 4. DroneX Usage (Tools → DroneX)

### 4.1 Basic Operation

1. Enter **Tools → DroneX** from main menu — auto-starts BLE scan and WiFi promiscuous sniffing. Phones can simultaneously connect to the device hotspot to view the web interface.
2. National Standard drones and old-format drones **appear in the same target list**, auto-merged by MAC — no mode-switching needed.

### 4.2 What Happens After Successful Identification

- **Buzzer alert**: Two quick "beeps" when new targets are found; dedicated alerts for blacklisted targets / emergency states.
- **Screen target page**: Shows model (e.g. "DJI Mavic 4 Pro", auto-inferred from serial number), serial number, position, altitude, speed, heading, pilot/takeoff point location, etc.
- **Serial**:

```
[NEW] DJI Mavic 4 Pro c4:d1:xx:xx:xx:xx via-WiFi(GB) rssi=-62
[HB] up=120 clients=0 rid=130 ble=12 wifi=128 dji=0 gb=130 sniff=1 ...
```

  - **`gb=N` (New)**: National Standard frame count; `wifi=` is old ASTM count; `dji=` is old DJI count.
- **Web JSON (new `"gb"` field)**: `"gb":130` in the device status interface = National Standard frame count. Can be used to build custom dashboards or integrations.
- **SD Card trajectories**: National Standard targets auto-create trajectory sessions and record waypoints — identical to old format behavior.
- **Black/White lists**: National Standard serial number (item 001) and the last 8 digits of real-name registration number (item 002) both participate in list matching.

### 4.3 WiFi Channel Strategy (Affects Detection Range/Probability)

DroneX needs to retain the phone hotspot service. When idle, it rotates **250ms on channel 1 + 250ms on channel 11 ↔ 500ms dwell on hotspot channel 6**. When phones are connected to the hotspot, it locks to channel 6. If the drone's 2.4G broadcast happens to be on another channel, detection will be slower. For full-channel (1~13) fast sweeping, use **RID Scanner**.

### 4.4 Serial Dump Output

```
[gb-dump] reason:-5 rssi:-70 len:83
FA 0B BC 0D 12 FF 20 48 FF FF FE ...
```

Same meaning as RID Scanner (error codes in Section 6). Max 1 per second, ring buffer up to 6 entries.

---

## 5. National Standard Data Field Reference (What the Device Shows)

| Standard Item | Content | Device Usage |
|---|---|---|
| 001 | Unique product identifier (SN, 20 bytes) | Target ID, brand/model inference, black/white lists |
| 002 | Last 8 digits of real-name registration number (UIN) | DroneX operator ID display, list matching |
| 004 | Drone classification (micro/light/small/medium/large) | Decoded and preserved |
| 005~007 | Control station / takeoff point coordinates and altitude | Pilot position (DroneX target details) |
| 008 | Drone coordinates | Target position, map, distance & bearing |
| 009 | Track angle (0.1°) | Heading |
| 010 | Ground speed (0.1 m/s) | Horizontal speed |
| 011 | Altitude relative to takeoff point (0.5m) | Relative altitude |
| 012 | Vertical speed (0.5 m/s) | Climb/descent rate |
| 013/014 | Geodetic altitude / barometric altitude | Absolute altitude |
| 015 | Operational status | Ground / air / emergency / RID failure alert |
| 020 | Unix millisecond timestamp | Time validation |

> National Standard operational status: 1=ground, 2=air, 3=emergency, 4/5=RID invalid; the device internally treats 4/5 uniformly as "system failure" and triggers alerts.

---

## 6. Dump Error Code (reason) Meaning

| reason | Meaning | Field Diagnosis |
|---|---|---|
| -1 | Parameter/length error | Incomplete frame, usually signal loss causing missing bytes |
| -3 | Type/version mismatch | Not a National Standard packet or unrecognized version |
| -4 | Data length field mismatch | Encapsulation difference or truncation |
| -5 | Required field missing / SN contains non-printable characters / value out of range | Most common: new model encapsulation differs. Please forward the hex dump |

If `reason:-5` appears consistently with a newly released drone model, save the full serial `[rid-dump]`/`[gb-dump]` hex output — it can be used to quickly add compatibility.

---

## 7. Real-World Test Procedure

1. Flash firmware (Section 2 partition settings must be correct), open serial monitor at 115200.
2. **Outdoors, open sky** — National Standard RID only broadcasts after the drone acquires GPS fix; indoors/no satellite = no broadcast.
3. Use a **domestic version** drone and DJI App. Overseas firmware / overseas location will broadcast ASTM F3411 instead.
4. Drone appears in the list within ~1 second after power-on (National Standard requires broadcast interval ≤1 second).
5. RID Scanner: watch header `GB:` grow; DroneX: watch heartbeat `gb=` grow and model names appear in the target list.
6. If no targets but FAIL/dump counts are rising, save the hex per Section 6 and forward.

---

## 8. Current Capability Boundaries (Important)

- **2.4 GHz WiFi beacon channel only**. ESP32-S3 hardware does not support 5.8 GHz sniffing; if new firmware only broadcasts RID on 5.8G, this device cannot receive it.
- **BLE 5.0 extended broadcast not yet supported**. Full National Standard packet is ~78 bytes, requiring BLE5 extended broadcast. Current NimBLE 1.4.2 can only receive traditional 31-byte broadcast. This will open after future upgrades to arduino-esp32 3.x + NimBLE 2.x (requires re-doing `libnet80211` symbol weakening).
- National Standard RID has **regional and geofencing gating** (see test conditions in Section 7).
- Recognition relies on WiFi Vendor IE — no SSID guessing, to avoid false positives from regular hotspots.

---

## 9. Technical Appendix

Files added/modified (all in project directory):

| File | Description |
|---|---|
| `opendroneidcn.h` / `opendroneidcn.cpp` | **New**. National Standard decode library, ported from OpenDroneID official `libopendroneidcn` (Apache-2.0). Receive-decode only, shared by both functions |
| `RIDScanner.cpp` | WiFi Vendor IE branch prioritizes National Standard identification; `GB:` counter; ring-buffered dump |
| `dronerx/dronerx_types.h` | New `GbMsg` structure, `g_gbQueue`/`g_gbPktCount` declarations |
| `dronerx/dronerx_wifi_rx.h` | National Standard identification and decode dispatch; fixed National Standard packets being silently dropped as misidentified ASTM packets; dump buffer |
| `dronerx/dronerx_impl.h` | National Standard queue consumption, `applyGb()` field mapping, web JSON / heartbeat counters |

National Standard packet identification signature (for technical reference):

```
WiFi Beacon → Vendor IE (0xDD):
  FA 0B BC 0D [counter] FF 2x [DataLen] [≥1 flag byte] [21 data items ...]
                         └─ Type 0xFF, version high 3 bits = 001 (0x20)
```

Protocol standard: GB 46750—2025 (available for download on Civil Aviation Administration website);
Decoder source: https://github.com/opendroneid/opendroneid-core-c/tree/master/libopendroneidcn

---

## 10. DroneX v2.0.0 Brand Upgrade (2026-10-15)

DroneX module upgraded from v1.10 to **v2.0.0**, unified name from "DroneRX Receiver" to "DroneX Transceiver", and National Standard RID support integrated simultaneously. User-visible changes below:

### 10.1 Boot Screen (Bilingual)

```
DroneX v2.0.0
大疆无人机 RID 收发器
DJI Drone RID Transceiver
```

- Changed from "DroneRX v1.9.0" to "DroneX v2.0.0"
- Single-line subtitle changed to bilingual two lines; function positioning changed from "Receiver" to "Transceiver" (has both RID detection and RID simulation capability)

### 10.2 Name Unification Table

| Location | Before Upgrade | After Upgrade |
|---|---|---|
| Boot screen title | DroneRX v1.9.0 | DroneX v2.0.0 |
| Boot screen subtitle | 大疆无人机 RID 接收器 | DJI Drone RID Transceiver (bilingual) |
| Info page version | v1.10 | v2.0.0 |
| Self-hosted hotspot SSID | DroneRX-xxxx | **DroneX-xxxx** |
| BLE device name | DroneRX | **DroneX** |
| Web title/Logo/loading page | DroneRX | DroneX |
| Web subtitle | Remote ID 接收器 | Remote ID Transceiver |
| License page title (web/screen) | DroneRX 授权 | DroneX 授权 |
| Serial startup banner | DroneRX v1.9 ... | DroneX v2.0.0 ... (ASTM + GB46750 National Standard) |

### 10.3 Usage Impact (Important)

- **Phones must reconnect to hotspot**: After upgrade, hotspot name is `DroneX-xxxx` (xxxx = last 2 hex digits of MAC). Previously saved `DroneRX-xxxx` will no longer appear.
- **BLE scan**: BLE device name now shows as `DroneX`.
- **Unchanged**: Hotspot password still `dronerx88`; mDNS domain still `dronerx.local`; web address `http://192.168.4.1` unchanged; SD card, black/white lists, license file (`dronex.lic`) and all data/functions unaffected.
- License flow unchanged: Unauthorized web auto-redirects to `/license`, license page title shows "DroneX License Activation".

### 10.4 Affected Files

`dronerx/dronerx_ui.h` (boot screen, info page version), `dronerx/dronerx_impl.h` (SSID, BLE name, banner), `dronerx/dronerx_web.h` (web UI), `dronerx/dronerx_license.h` (license page), `dronerx/pc/dronerx_pc.html` (PC management console).

---

## 11. RID Transmitter (New 2026-10-15, Tools → RID Transmitter)

### 11.1 Feature Overview

"RID Transmitter" turns the device into a multi-drone signal source, **simultaneously broadcasting up to 12 (default 10) distinct identities**. Unlike the DroneX built-in simulator, this function is a **multi-protocol broadcast transmitter** — used to test RID receiver device/software recognition capability.

Each identity = one independent BSSID/SA 802.11 Beacon frame, which mounts **three protocol Vendor IEs (Element ID 221 = 0xDD) in parallel**:

| Protocol | Vendor IE Content | Description |
|---|---|---|
| ASTM F3411-22a | `FA 0B BC 0D <counter> <MessagePack>` | International standard RID; also transmits `90:3A:E6` old OUI variant |
| GB 46750-2025 National Standard | `FA 0B BC 0D <counter> FF 20 <len> <3-byte identifier chain> <21 data items>` | Format used by new domestic-firmware DJI |
| DJI Proprietary DroneID | `26 37 12 58 62 13 <70-byte flight_info>` | Old Enhanced-WiFi structure (arXiv:2207.10795 Fig.3) |

Since each identity has a different BSSID, **one round (millisecond-level) of N Beacon transmissions is recognized by receivers as N drones** — implementing "simultaneous multi-RID per transmission".

BLE side has a separate ASTM message channel: **120ms time-slice rotation** broadcasting 31-byte raw packets `[30][0x16][FA FF][0D][counter][25-byte message]` per identity (UUID 0xFFFA service data). Since one radio has only one broadcast address, BLE can only rotate N random-address identities in time slices.

### 11.2 Operation

1. Enter **Tools → RID Transmitter** from main menu — auto-starts transmission on entry (RUN state).
2. Screen shows: run state, channel, identity count, transmit cycle count, WiFi success/failure counts, per-protocol switches, WiFi/BLE frame counts, summary of first two identities (model, SN, coordinates).
3. Press **SEL key** to exit — auto-stops transmission and releases RF on exit.
4. Serial monitor (115200) outputs heartbeat every 2 seconds, with real-time command control.

### 11.3 Serial Commands

| Command | Action |
|---|---|
| `start` / `stop` | Start / pause transmission |
| `status` | Print summary counts (cycles, WiFi/BLE success-failure, per-protocol switches) |
| `list` | Print all identity details (model, SN, UIN, position, BSSID, per-protocol counter) |
| `n <1..12>` | Re-generate specified number of identities |
| `regen` | Re-randomize identities with current count |
| `pos <lat> <lon> [alt]` | Modify start point and re-distribute drone positions (default Shanghai People's Square 31.2304, 121.4737) |
| `wifi <0\|1>` | WiFi Beacon master switch (all three Vendor IEs go over WiFi) |
| `astm <0\|1>` | ASTM F3411 Vendor IE switch (includes 90:3A:E6 variant) |
| `gb <0\|1>` | GB 46750 National Standard packet switch |
| `dji <0\|1>` | DJI Proprietary DroneID Vendor IE switch |
| `ble <0\|1>` | BLE time-slice broadcast switch |
| `exit` | Exit this function (same as SEL key) |
| `help` | Show command help |

### 11.4 Self-Transmit-Self-Receive Verification (Closed-Loop Test)

1. Confirm heartbeat `wifi_ok` keeps growing in "RID Transmitter";
2. Exit and enter **RID Scanner**: should see N targets with different MACs. Watch header `BCN / VIE / DJI / GB` counts grow with broadcasts;
3. Or enter **DroneX**: target list shows multiple drones, details page shows **DJI Mini 2 / DJI Mavic Air 2**, etc. (SN/position/altitude verifiable item by item);
4. Can also use another ESP32-DIV, phone RID App, or DroneSecurity (SDR) as the receiver for cross-verification.

### 11.5 Model & Serial Number Rules

- Identity model is randomly drawn from a built-in model code table (this update adds **DJI Mini 2 = 3Q4** and **DJI Mavic Air 2 = 3N3**; transmitter and receiver model tables are synchronized to ensure self-transmit-self-receive displays correct model names).
- RID serial number = `1581F` + 3-digit model code + random fill, total 20 characters — matches DJI's official "14-digit body SN model: `1581F` + 14-digit SN + `0`" rule (Mini 2 / Mavic Air 2 are this type).

### 11.6 Capability Boundaries (Important)

- **DJI Mini 2 / Mavic Air 2 are OcuSync 2.0 models** (DJI official: Mini 2 / MA2 → O2). ESP32 **cannot transmit real OcuSync RF**. This function transmits **802.11-borne RID frames** for these model identities (ASTM + National Standard + DJI three protocols in parallel) — for 802.11 RID receiver devices to recognize.
- RUB-SysSec/DroneSecurity is an open-source implementation for **receiving/demodulating DJI DroneID** (SDR decoding OcuSync) — complementary to this function. This device handles the "transmit" side; you can use its decoding results to verify frame structure.
- DJI `flight_info` inner field offsets are not fully public; current layout follows the field order given in arXiv:2207.10795 (version/SN/status/SN/coord-alt/speed/yaw). Once real-device captures are available, **only `buildDjiPayload()` in `dronerx_ridtx.h` needs modification** to correct.
- Model codes `3Q4` / `3N3` are taken from FAA registration public samples (`3Q4SJ1E00357LX` / `3N3BHAM012022G`) — **not yet verified against CAAC "DJI Drone Model Code Table" or real captures**. Marked "pending verification".
- One BLE radio has only one broadcast address, so the BLE channel is time-slice rotated (120ms per identity). External scanners see a target refresh interval of N × 120ms.

### 11.7 Affected Files

| File | Description |
|---|---|
| `dronerx_ridtx.h` | **New (project root, independent from DroneX)**. RID multi-protocol transmitter: identity model, ASTM/GB/DJI three encoders, WiFi raw Beacon transmission (reuses existing `esp_wifi_80211_tx` capability), BLE time-slice rotation, serial commands, screen status page |
| `ESP32-DIV-1.8.1.ino` | Tools sub-menu added item #6 "RID Transmitter" (index + icon synchronized), dispatches `RidTX::setup/loop/exit` |
| `dronerx/dronerx_types.h` | Receiver model table `djiModelFromSN()` adds `3Q4 → DJI Mini 2`, `3N3 → DJI Mavic Air 2` |
| `dronerx/dronerx_sim.h` | DroneX simulator model table `SIM_MODELS` synchronously adds the two models above |
