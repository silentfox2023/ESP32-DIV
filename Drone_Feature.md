# DroneX and Drone (Remote ID) Feature Guide

**Firmware:** ESP32-DIV v1.9.3 · **DroneX:** v2.1.0 · **Hardware:** ESP32-DIV V2 (ESP32-S3, 16 MB flash)

This guide covers the two drone-related layers of the firmware:

| Layer | Location | Purpose |
| --- | --- | --- |
| **Drone** | `More > Drone` | Field tools: RID Scanner, RID Spoof, Drone Deauth, Drone Jelly |
| **DroneX** | Main menu | Continuous detection, tracking, alerts, lists, SD tracks, browser dashboard, simulator, and a Wi-Fi deauth lab |

All drone tools decode **ASTM F3411**, **GB 46750-2025**, and **DJI proprietary DroneID**. Use them only on equipment you own or are explicitly authorized to test.

---

## 1. Access and licensing

### 1.1 Drone layer

The whole **More > Drone** layer is gated by a device MAC whitelist (`rid_feature_lock.h`). The entry is always visible and can always be opened; on an unauthorized device it shows a single English license notice page instead of the tool list. The check is made once when the layer opens, so the four tools inside are not checked again. Press **LEFT** / **SELECT**, or tap **Back**, to return.

Authorization is bound to the Wi-Fi STA MAC address and is independent of the DroneX license.

### 1.2 DroneX

DroneX requires a paid license. The device ID (Wi-Fi STA MAC address) is shown on the lock screen and in the browser. A license is bound to that MAC address and signed with ECDSA over secp256r1; only the public key is embedded in the firmware, so copying a license to another device fails verification.

| Method | Procedure |
| --- | --- |
| Web | Join the DroneX hotspot, open `http://192.168.4.1`, paste the Base32 license, submit |
| SD card | Save the license as a plain ASCII/UTF-8 text file named `dronex.lic` in the card root, then power on with the card inserted |
| Serial | Send `DRONEXKEY,<license>` at 115200 bps |

A successful activation is stored permanently and revalidated on every boot. The lock screen returns to ESP32-DIV after 5 minutes, or immediately on a SELECT long press.

---

## 2. More > Drone

### 2.1 RID Scanner

**Open:** `More > Drone > RID Scanner`.

The scanner starts automatically and alternates between:

- **Wi-Fi phase:** 6.5 seconds, one full sweep of channels 1-13 at 500 ms per channel.
- **BLE phase:** 5 seconds, with the Wi-Fi promiscuous interface paused so BLE gets the radio.

The radio is time-multiplexed because Wi-Fi promiscuous mode and BLE scanning share the same 2.4 GHz radio; running both at once causes heavy packet loss.

#### Screen

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

- **`DRN` / `ON`:** targets in the list / targets currently online.
- **`pkt`:** RID packet count (NAN + RID beacons + BLE ODID).
- **`MGMT`:** management frames seen.
- **`BLE`:** BLE advertisement callbacks.
- **`ODID`:** BLE ODID messages decoded.
- **`NAN`:** Wi-Fi NAN RID broadcasts.
- **`BCN`:** RID beacons.
- **`DJI`:** DJI proprietary DroneID frames.
- **`FAIL`:** suspected RID frames that failed parsing.
- **`MGB`:** all beacons observed.
- **`VIE`:** vendor information elements observed.
- **`GB`:** GB 46750-2025 frames decoded successfully.
- **`IF` (interface):** 1 = Wi-Fi beacon, 2 = Wi-Fi NAN, 3 = BLE 4 legacy advertising, 4 = BLE 5 extended advertising.

#### Channel modes and controls

| Control | Action |
| --- | --- |
| `LEFT` / `RIGHT` | Step the locked channel; pressing either while in AUTO switches to LOCK |
| `DOWN` | Toggle AUTO / LOCK |
| `UP` | Page the target list (3 rows per page) |
| `SELECT` | Exit |
| Touch bar | `Ch-`, `Lock`/`Auto`, `Exit`, `Page`, `Ch+` |

- **AUTO** (default) sweeps channels 1-13.
- **LOCK** stays on one channel for a higher hit rate; the default lock channel is 6.
- On exit the channel is reset to `AUTO / CH1` so the next feature starts clean.
- While locked, targets on other channels are naturally invisible.

#### Target lifetime

- No advertisement for **5 seconds** → target turns gray.
- No advertisement for **30 seconds** → target is removed.

#### Low-heap / Wi-Fi failure behavior

If the Wi-Fi driver cannot start, or the heap gate refuses the Wi-Fi allocation, the scanner continues in **BLE-only** mode. The screen reports `BLE only (WiFi init failed)` or `BLE only (LOW HEAP)` with the free and required heap values. BLE targets are still decoded.

---

### 2.2 RID Spoof

**Open:** `More > Drone > RID Spoof`.

RID Spoof turns the device into a virtual drone fleet. It transmits **10 identities by default and up to 12**, each with a unique BSSID/MAC, serial number, operator ID, model, position, altitude, speed, and heading. It emits RID frames; it does not emulate OcuSync or Lightbridge radio links.

#### Protocols

| Bearer | Protocol | Notes |
| --- | --- | --- |
| Wi-Fi beacon on channel 6 | ASTM F3411 | Includes the `FA:0B:BC` and `90:3A:E6` OUI variants |
| Wi-Fi beacon on channel 6 | GB 46750-2025 | Domestic Chinese standard |
| Wi-Fi beacon on channel 6 | DJI proprietary DroneID | OUI `26:37:12` |
| BLE legacy advertising | ASTM F3411 service data `0xFFFA` | 31-byte advertisement, 120 ms time slice per identity |

One Wi-Fi beacon round is millisecond-level, so a receiver sees N distinct drones. BLE has one broadcast address, so N identities are rotated in time slices.

#### Placement

- Default start point: `31.2304160, 121.4737010`, altitude 50 m, speed 3 m/s, heading 45°. Change it with `pos <lat> <lon> [alt]`.
- Aircraft are spread around the start point at 300 m and increase by 80 m per aircraft.
- Altitude, horizontal speed, and heading increase per aircraft.
- The remote station is placed 150 m outside each aircraft.
- Identities are randomized on boot and include model names such as DJI Mini 2 and DJI Mavic Air 2.

#### Screen and controls

```text
RID SPOOF
[ RUN ] [ STEADY ] [ DEF ]
PROTO ASTM+GB+DJI  CH6  BLE ON
DRN 10  PKT 4820                         96/s
Transmitting  12s

IDENTITY  MODEL / SN                       1/4
#01 DJI Mavic Air 2
SN 1581F3N3...

[ START ] [ STEADY ] [ BLE ON ]
```

| Control | Action |
| --- | --- |
| `START` / `STOP` button, `SELECT` | Start or stop transmission |
| `STEADY` / `FLOOD` button, `RIGHT` | 1000 ms or 250 ms Wi-Fi burst period |
| `BLE ON` / `BLE OFF` button | Enable or disable the BLE channel |
| `Proto` / `DOWN` | Cycle `DEF` → `ASTM` → `GB` → `DJI` |
| `Page` / `UP` | Page the identity list |
| `Back` / `LEFT` | Exit and release the radio |

The protocol preset only changes the Wi-Fi vendor-IE combination. BLE is controlled separately.

#### Serial commands

| Command | Action |
| --- | --- |
| `start` / `stop` | Start or pause transmission |
| `status` | Print the summary counts |
| `list` | Print all identity details |
| `regen` | Re-randomize the fleet with the current count |
| `n <1..12>` | Set the number of identities |
| `pos <lat> <lon> [alt]` | Move the fleet start point |
| `wifi <0\|1>` | Master switch for all Wi-Fi RID output |
| `astm <0\|1>` | ASTM vendor IE switch |
| `gb <0\|1>` | GB 46750 packet switch |
| `dji <0\|1>` | DJI proprietary DroneID switch |
| `ble <0\|1>` | BLE time-slice switch |
| `exit` | Leave RID Spoof |
| `help` | Show command help |

#### Boundaries

- ESP32 hardware transmits **802.11-borne RID frames** only. It cannot transmit real OcuSync or Lightbridge links.
- Model codes for DJI Mini 2 and Mavic Air 2 follow public FAA registration samples and have not been verified against CAAC model tables or real captures.
- Wi-Fi output is fixed to channel 6; a receiver locked to another channel will not see it.
- BLE uses legacy 31-byte advertising, not BLE 5 extended advertising.

---

### 2.3 Drone Deauth

**Open:** `More > Drone > Drone Deauth`.

This tool passively identifies Wi-Fi drone links in 2.4 GHz management frames, then sends 802.11 deauthentication frames to selected targets. Use it only on authorized equipment and networks.

#### Target identification

| Evidence | Classification |
| --- | --- |
| RID vendor IE `FA:0B:BC` / `90:3A:E6` (ASTM F3411 and GB 46750) | `RID` |
| DJI DroneID vendor IE `26:37:12` | `DJI` |
| NAN / Public Action RID broadcast (`51:6F:9A:01:00:00`) | `NAN` |
| SSID prefix such as `DJI-`, `TELLO-`, `MAVIC-` | `SSID` (weak evidence only) |

The scanner hops channels 1-13 every 300 ms while it is not attacking and keeps up to 16 targets, with up to 4 observed clients per target. A target expires after 30 seconds without a frame.

#### Attack behavior

- The device impersonates the target BSSID/SA.
- If a client is observed on the target, three out of four frames are sent to that client and one is broadcast.
- If no client is observed, all frames are broadcast.
- The reason code is 7 (class 3 frame).
- Frames are sent every 10 ms while attacking.
- This only affects 802.11 links, such as a drone's own access point or a controller app associated with it. It has no effect on OcuSync, Lightbridge, or other proprietary frequency-hopping links, and it cannot reach 5 GHz.

#### Screen and controls

| Control | Action |
| --- | --- |
| `UP` / `DOWN` | Select the previous / next target |
| `SELECT` | Arm or disarm the selected target |
| `RIGHT` | Start or stop the attack; if nothing is armed, all targets are armed first |
| `LEFT` | Exit |
| Tap a row | Select and arm/disarm that target |
| Touch bar | `Back`, `Next`, `Arm`, `Prev`, `ATK`/`Stop` |

The status box shows `LISTENING <n> drones <n> links` while idle and `ATTACKING <armed>/<total> armed` while transmitting. The counters show the current channel, total frames sent, and frames per second.

---

### 2.4 Drone Jelly

**Open:** `More > Drone > Drone Jelly`.

Drone Jelly uses the three nRF24 modules on the same channel to transmit a continuous 2.4 GHz carrier or sweep aimed at drone control and video links.

| Mode | Channels | Frequency |
| --- | --- | --- |
| **BARRAGE** | 0-125 | 2400-2525 MHz |
| **FOCUS** | 20-80 | 2420-2480 MHz |

All three modules transmit on the same channel for higher combined output. The feature refuses to start while another nRF24 feature still owns the shared SPI bus and displays `BUS BUSY` instead.

| Control | Action |
| --- | --- |
| `UP` or `SELECT` | Start / stop jamming |
| `RIGHT` | Toggle BARRAGE / FOCUS |
| `LEFT` | Exit |
| Touch bar | `Back`, `Start`/`Stop`, `Mode` |

On exit the carrier is stopped, the radios are powered down, the shared SPI bus is released, and the SD card path is restored.

---

## 3. DroneX

DroneX is a separate firmware system for continuous monitoring. It time-slices BLE and Wi-Fi Remote ID reception, merges observations into stable targets, and provides on-device and browser-based management.

### 3.1 Entry and exit

- Select the **DroneX** tile on the main menu. The firmware stores DroneX as the boot system and reboots into it. There is no in-place return to the main menu.
- Leave DroneX from **Info > Exit DIV**, or hold **SELECT** for 1.5 seconds. The device reboots back into ESP32-DIV.
- If the license is missing, the lock screen is shown instead. It returns to ESP32-DIV after 5 minutes, or immediately on a SELECT long press.

### 3.2 Tabs

| Tab | Purpose |
| --- | --- |
| **Scan** | Live target cards, signal strength, target count |
| **Setup** | Language, buzzer, backlight, observer point, router STA, reboot |
| **Info** | Uptime, heap, clients, receive counters, SD self-test, 5-second manual scan, Exit DIV |
| **Sim** | 1-3 virtual targets, motion pattern, start/stop |
| **Atk** | Wi-Fi deauth lab (AP/client scan, target selection, attack) |

Navigation:

- `LEFT` / `RIGHT`: previous / next tab.
- `UP` / `DOWN`: move the selection (Scan, Setup, Sim, Atk as applicable).
- `SELECT`: open the selected target or perform the page action.
- Touch: tap tabs, cards, buttons, and list rows.

### 3.3 Detection

- Decodes **ASTM F3411**, **GB 46750-2025**, and **DJI proprietary DroneID**.
- Merges BLE and Wi-Fi observations of the same aircraft by MAC.
- Keeps up to **6 live targets**. The Scan tab shows 3 cards; if more targets are present, it displays `+N more, see web 192.168.4.1`.
- A target turns gray after **8 seconds** without an advertisement and is removed after **30 seconds**.
- Target details include model, ordinal, status, serial number, RSSI, BSSID, latitude, longitude, altitude, speed, distance, and operator position.

### 3.4 Alerts, lists, and logs

- A new target triggers an audible alert; blacklisted, emergency, and fault states have dedicated alarms on screen and in the browser.
- Black and white lists match serial number, real-name registration number (last 8 digits), or MAC prefix.
- Each target creates an SD track session:
  - `/dronerx/history.csv` — history index
  - `/dronerx/tracks/<sessionId>.csv` — track points
- A missing or unmounted SD card disables history and track logging without stopping detection.

### 3.5 Simulator

The **Sim** tab and serial commands inject virtual targets for demonstrations and training:

| Control | Action |
| --- | --- |
| Start / stop | Start or stop the simulator |
| Count | 1-3 simulated targets (default 3) |
| Motion pattern | Cycle the movement pattern |
| Swap | Switch the simulated identity set |

Serial equivalents: `SIM START`, `SIM STOP`, `SIM SWAP`, `SIM PAT`, `SIM N <1-3>`.

The simulator injects into the same target list as real traffic but does not replace or modify real detections.

### 3.6 Setup

| Item | Function |
| --- | --- |
| Language | Switch the on-device UI between Chinese and English |
| Buzzer Alert | Enable or disable the detection beep |
| Backlight | Turn the display backlight on or off |
| Home Point | `Clear`, `GPS`, or `Manual` latitude/longitude entry |
| Router WiFi (STA) | Edit SSID/password, `Connect`, `Disconnect`, `Clear` |
| Reboot | Restart the device |

The home point is persisted in NVS and survives reboots. The default is `26.4875092, 106.7232399`.

### 3.7 Info

The Info tab shows uptime, free heap and largest block, AP client count, BLE/RID/Wi-Fi counters, live targets, and the current channel. It also provides:

- **SD Test** — real write / read-back / delete self-test with the failing step and free space.
- **Scan 5s** — manual full-channel sweep (1-13, about 350 ms each). It is only available while STA is not connected; AP clients may briefly drop.
- **Exit DIV** — reboot back to ESP32-DIV.

### 3.8 Browser dashboard

| Setting | Default |
| --- | --- |
| Hotspot SSID | `DroneX-xxxx` (last two hex digits of the STA MAC) |
| Hotspot password | `dronerx88` |
| Dashboard URL | `http://192.168.4.1` |
| mDNS name | `http://dronerx.local` (started only when internal heap is sufficient) |

The root page offers a mobile management UI (`/m`) and a PC management UI (`/pc`); the choice is remembered. The dashboard shows live statistics, the target list, a map or offline tactical view, alerts, history, and editable lists and settings.

**AP+STA:** the DroneX hotspot is always kept on and anchored to channel 6 when STA is not connected, while the device may also join a router. In STA mode the single radio is locked to the router channel, so Wi-Fi RID is received only on that channel; BLE detection continues. The firmware does not shut the hotspot down in STA mode.

### 3.9 Wi-Fi channel strategy

The Wi-Fi sniffer uses a single radio with this schedule:

| Condition | Behavior |
| --- | --- |
| STA connected to a router | Radio locked to the router channel; no hopping |
| AP client connected, no STA | Stays on the AP anchor channel (channel 6), with grace periods to avoid dropping the web UI |
| Idle, RID seen in the last 20 s ("warm") | 6 s scan window: anchor channel 900 ms + two alternate channels (channel 6 plus 1/11, depending on the anchor) 250 ms each; then 4 s AP window on the anchor |
| Idle, no RID for 20 s ("cold") | 6 s scan window: channels 1-13, with 800 ms on 1/6/11 and 300 ms on the others; then 4 s AP window on the anchor |

The 4-second AP window lets phones complete scanning, association, and the handshake without the AP hopping away. If a channel change is refused while a client is attached, the firmware may briefly deauthenticate AP clients once and retry; this is why a phone may need to reconnect after a forced channel move.

### 3.10 Serial commands

| Command | Action |
| --- | --- |
| `DRONEXKEY,<license>` | Activate DroneX |
| `HOME,<lat>,<lon>` | Set and persist the observer point |
| `HOME,CLEAR` | Clear the observer point and restore the default |
| `LIST` | Print the current target list |
| `SIM START` / `SIM STOP` | Start or stop the simulator |
| `SIM SWAP` / `SIM PAT` | Change identities or motion pattern |
| `SIM N <1-3>` | Set the simulated target count |
| `STA,<ssid>,<pass>` | Join a router; the AP stays on |
| `STA,DISCONNECT` / `STA,CLEAR` | Disconnect or clear the STA credentials |
| `VERBOSE,1` / `VERBOSE,0` | Enable or disable detailed logs |
| `help` | Show DroneX-specific commands |

Heartbeat example:

```text
[HB] up=120 clients=0 rid=130 ble=12 wifi=128 dji=0 gb=130 sniff=1 ...
[NEW] DJI Mavic 4 Pro c4:d1:xx:xx:xx:xx via-WiFi(GB) rssi=-62
```

- `rid` — accepted RID messages.
- `ble` — BLE advertisements processed.
- `wifi` — legacy ASTM Wi-Fi messages.
- `dji` — legacy DJI messages.
- `gb` — GB 46750 messages.
- `sniff` — Wi-Fi sniffer state (1 while the task is running, including STA mode).
- `ch` — current Wi-Fi channel.

The web status JSON also exposes `"gb": <count>`.

### 3.11 Atk tab (Wi-Fi deauth lab)

The **Atk** tab is a general 802.11 deauthentication lab, not a drone-only tool:

1. Turn **Scan ON** at the top right. The firmware scans nearby APs and associated clients in promiscuous mode.
2. Tap a row to select a target (AP or client); it stays in a fixed slot so the list does not reorder.
3. Press **SELECT** to start or stop the attack, or double-tap the list area.
4. Targets expire after 15 seconds without frames while scanning is on.

The attack sends 802.11 deauth frames every 100 ms on the target's real channel. Use it only on networks you own or are authorized to test.

---

## 4. GB 46750-2025 reference

### 4.1 Standard fields

| Item | Content | Device usage |
| --- | --- | --- |
| 001 | Unique product identifier (20-byte serial number) | Target ID, model inference, black/white lists |
| 002 | Last 8 digits of the real-name registration number (UIN) | Operator ID display, list matching |
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

Operational status mapping: 1 = ground, 2 = air, 3 = emergency, 4/5 = RID invalid. The device treats 4/5 as a system failure and raises an alert.

### 4.2 Frame signature

```text
Wi-Fi Beacon → Vendor IE (0xDD):
  FA 0B BC 0D [counter] FF 2x [DataLen] [flags] [21 data items ...]
                         └─ type 0xFF, version high 3 bits = 001 (0x20)
```

GB broadcasts reuse the ASD-STAN vendor OUI but carry a different body. The decoder requires the GB type/version and strict field validation, so a normal Wi-Fi hotspot is not misidentified.

### 4.3 Diagnostic dumps

If a GB frame has a matching header but fails field validation, the firmware prints the raw frame (at most one per second):

```text
[rid-dump] reason:-5 rssi:-64 ch:6 mac:C4:D1:xx:xx:xx:xx ie_len:83
FA 0B BC 0D B0 FF 20 48 FF FF FE 2A 31 35 38 31 46 ...
```

DroneX uses the same data with a `[gb-dump]` tag:

```text
[gb-dump] reason:-5 rssi:-70 len:83
FA 0B BC 0D 12 FF 20 48 FF FF FE ...
```

| reason | Meaning | Diagnosis |
| --- | --- | --- |
| -1 | Parameter / length error | Incomplete frame, usually signal loss |
| -3 | Type / version mismatch | Not a GB packet or an unsupported version |
| -4 | Data length mismatch | Encapsulation difference or truncation |
| -5 | Required field missing, non-printable serial, or value out of range | Most common for a new model; save the hex dump |

If `reason:-5` repeats for a newly released drone model, save the full `[rid-dump]` / `[gb-dump]` output; it can be used to add compatibility.

---

## 5. Field test procedure

1. Flash firmware v1.9.3 and open the serial monitor at 115200 bps.
2. Test **outdoors with open sky**. GB 46750 RID is transmitted only after the aircraft obtains a GNSS fix; indoors or without a fix there may be no broadcast.
3. Use a domestic-firmware drone and the DJI app. Overseas firmware or overseas locations normally broadcast ASTM F3411 instead.
4. Power on the aircraft. GB requires a broadcast interval of 1 second or less, so a target should appear within about 1 second.
5. **RID Scanner:** watch the `GB` counter grow. `FAIL` growing without `GB` means the frame header matches but decoding fails.
6. **DroneX:** watch the `gb=` heartbeat counter grow and the model name appear in the target list.
7. **Closed-loop transmitter test:** start RID Spoof, then open RID Scanner or DroneX. RID Scanner should show N targets with different MACs and growing `BCN` / `VIE` / `DJI` / `GB` counters. DroneX should show multiple drones with model names and positions.
8. If no targets appear but `FAIL` or dump output grows, save the hex dump from step 6.

---

## 6. Known limitations

- **2.4 GHz only.** ESP32-S3 hardware cannot receive or transmit 5.8 GHz RID.
- **BLE 5 extended advertising is not decoded.** Only legacy 31-byte BLE advertising is used.
- **No OcuSync / Lightbridge emulation.** RID Spoof and Drone Deauth operate on 802.11 frames only.
- **GB gating.** GB 46750 broadcasts depend on a GNSS fix, the region, and the aircraft firmware.
- **Single radio.** Wi-Fi and BLE features are mutually exclusive while active. In DroneX STA mode the radio is locked to the router channel, so full-channel Wi-Fi RID scanning requires the AP-only configuration.
- **DroneX capacity.** Up to 6 live targets; the Scan tab shows 3 cards and the browser shows the full list.
- **DroneX simulator.** 1-3 virtual targets; it does not replace real detections.
- **RID Scanner heap gate.** On a fragmented or low heap, the scanner may start in BLE-only mode.
- **Drone Deauth scope.** It affects 802.11 links only and cannot reach OcuSync, Lightbridge, or 5 GHz.
- **Drone Jelly sharing.** It cannot start while another nRF24 feature owns the shared SPI bus.
- **Authorized use only.** Deauthentication and jamming features must be used only on equipment and networks you own or are explicitly authorized to test.

---

## 7. Troubleshooting

**The Drone menu shows a license page.**
The device MAC is not in the Drone whitelist. Drone and HaleHound share this MAC whitelist; DroneX uses a separate ECDSA license.

**DroneX finds no drones indoors.**
RID requires an outdoor environment and a GNSS fix on the aircraft. Move outdoors and keep the device in AP mode for full-channel Wi-Fi RID reception.

**The DroneX hotspot is not visible after joining a router.**
Current firmware keeps the hotspot on in STA mode. The first phone association may be dropped once; reconnect. If it is still missing, reboot and check the Info tab for AP and client state.

**RID Scanner shows `BLE only`.**
The Wi-Fi driver failed to start or the heap gate rejected the Wi-Fi allocation. Close other radio features, reboot, and retry. The screen shows the free and required heap values.

**RID Spoof is not detected by a receiver.**
Check that the receiver is on channel 6, that the protocol preset includes the expected format (`DEF` enables ASTM + GB + DJI), and that the relevant switch is on (`wifi`, `astm`, `gb`, `dji`, or `ble`).

**DroneX heartbeat shows `sniff=0`.**
This is not normal in STA mode; promiscuous sniffing should stay enabled with the radio locked to the router channel. A persistent `sniff=0` points to a failed sniffer task or very low heap. Reboot and check the `[MEM]` and `[HB]` lines.

**A GB drone appears but its fields are empty or `FAIL` rises.**
The frame header was recognized but a mandatory field failed validation. Capture the `[rid-dump]` / `[gb-dump]` hex output for diagnosis.
