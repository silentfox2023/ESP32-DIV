Starting May 1, 2026, **GB 46750-2025 "Civil Unmanned Aircraft System Operation Identification Specification"** becomes mandatory. New DJI drones (such as the Mavic 4 Pro, Air 3S, and Mavic 3 series with updated firmware) have switched their WiFi broadcasts to the Chinese National Standard (GB) data format.

As a result, both **RIDScanner** and **DroneX** have been updated with GB decoding capabilities, featuring automatic format recognition.

---

## 1. RIDScanner User Guide (`Tools` -> `RIDScanner`)

### 1.1 Operation

1. Navigate to **`Tools` -> `RIDScanner**` from the main menu. The device will automatically start time-shared scanning for WiFi (sweeping channels 1–13) and BLE.
2. The screen refreshes approximately every 0.3 seconds. Press the **SEL key to exit**.
3. Detected drones without broadcasts for 5 seconds will turn gray, and will be automatically cleared after 30 seconds.

### 1.2 Screen Fields

```text
RID WIFI CH:6   ← Current phase (WIFI/BLE) and channel
Online:1 Total:1  ← Online count / Total count
MGMT:123 BLE:45 ODID:0  ← Management frames / BLE callback / BLE ODID count
NAN:0 BCN:5 DJI:0 FAIL:1 ← NAN / RID beacon / DJI proprietary / Parse failure
MGB:320 VIE:88 GB:3     ← Total beacons / Vendor IE / [GB count (New)]

C4D1 if:1 RSSI:-62      ← Last two bytes of MAC / Interface type / RSSI
ID:1581F8LQxxxxxxxx     ← Product serial number (GB Item 001)
31.23045, 121.47370     ← Drone latitude and longitude

```

* **`GB:n` (New)**: Number of successfully decoded GB RID frames. If this number increases with broadcasts (GB broadcast interval <= 1s), it indicates successful recognition.
* **`if:n` (Interface Type)**: 1 = WiFi Beacon (displayed by GB drones), 2 = WiFi NAN, 3 = BLE 4 Broadcast, 4 = BLE 5 Extended Broadcast.
* **`FAIL:n`**: Count of suspected RID frames that failed decoding (including cases where GB frame field validation failed).

### 1.3 Serial Diagnostic Output

When a GB frame has a matching header but fails field validation, the serial port automatically outputs the raw data (up to 1 entry per second) to help verify new model packaging:

```text
[rid-dump] reason:-5 rssi:-64 ch:6 mac:C4:D1:xx:xx:xx:xx ie_len:83
FA 0B BC 0D B0 FF 20 48 FF FF FE 2A 31 35 38 31 46...

```

---

## 2. DroneX User Guide (`Tools` -> `DroneX`)

### 2.1 Basic Operation

1. Navigate to **`Tools` -> `DroneX**` from the main menu. BLE scanning and WiFi promiscuous sniffing will start automatically, allowing your phone to connect to the device hotspot simultaneously to view the web interface.
2. GB drones and legacy format drones **appear in the same target list**, automatically merged by MAC address without needing to switch modes.

### 2.2 Behavior After Successful Recognition

* **Buzzer Alerts**: Two "da-da" beeps when a new target is discovered; corresponding warning sounds for blacklisted targets or emergency states.
* **Screen Target Page**: Displays aircraft model (e.g., "DJI Mavic 4 Pro", automatically inferred from the serial number), serial number, position, altitude, speed, heading, and pilot/takeoff location, etc.
* **Serial Output**:
```text
[NEW] DJI Mavic 4 Pro c4:d1:xx:xx:xx:xx via-WiFi(GB) rssi=-62
[HB] up=120 clients=0 rid=130 ble=12 wifi=128 dji=0 gb=130 sniff=1 ...

```


* **`gb=N` (New)**: GB frame count; `wifi=` is the legacy ASTM count, and `dji=` is the legacy DJI count.
* **Web JSON (`"gb"` field added)**: In the device status API, `"gb":130` represents the GB frame count, which can be used for custom dashboards or integrations.
* **SD Card Trajectory**: GB targets automatically establish trajectory sessions and record waypoints, exactly identical to legacy formats.
* **Black/White Lists**: Both the GB serial number (Item 001) and the last 8 digits of the real-name registration number (Item 002) participate in list matching.

### 2.3 WiFi Channel Strategy (Affects Detection Range/Probability)

To maintain the phone hotspot service simultaneously, DroneX cycles through **channels 1 and 11 for 250ms each ↔ dwelling on hotspot channel 6 for 500ms** during idle periods, and locks to channel 6 when a phone is connected to the hotspot. If a drone's 2.4G broadcast happens to be on another channel, discovery may be slower. Use **RIDScanner** when you need a fast, full-channel (1–13) search.

### 2.4 Serial Sample Capture Output

```text
[gb-dump] reason:-5 rssi:-70 len:83
FA 0B BC 0D 12 FF 20 48 FF FF FE ...

```
