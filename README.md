# Solarflow Web Bluetooth Manager

A browser-based tool to manage Zendure Solarflow devices over Bluetooth Low Energy (BLE) — no app, no Python, no Raspberry Pi required. Runs directly from your phone or laptop using the Web Bluetooth API.

Built as a single HTML file with zero dependencies (React loaded from CDN).

## Features

- **WiFi Scanning** — scan for nearby networks directly through the device, with signal strength and auth mode
- **WiFi Provisioning** — connect your Solarflow device to WiFi without the Zendure app
- **MQTT Redirection** — point the device at a local MQTT broker (offline mode) or back to Zendure cloud
- **Live Telemetry** — real-time dashboard with solar input, battery status, home output, pack data
- **Device Configuration** — output/input limits, min/max SoC, bypass mode, buzzer, inverter settings
- **Raw Property Viewer** — inspect all device properties as JSON
- **BLE Log** — full TX/RX log of all BLE communication for debugging

## Supported Devices

| Device | Product Key | BLE Name Prefix | Tested |
|---|---|---|---|
| Hub 1200 | `73bkTV` | `zenp` | By original bt-manager |
| Hub 2000 | `A8yh63` | `zenh` | By original bt-manager |
| AIO 2400 | `yWF7hV` | `zenr` | By original bt-manager |
| Hyper 2000 | `ja72U0ha` | `zene` | By original bt-manager |
| ACE 1500 | `8bM93H` | `zenf` | By original bt-manager |
| **SolarFlow 800 Pro** | `R3mn8U` | `zen` | ✅ This tool |

## Browser Compatibility

- **Chrome on Android** — full Web Bluetooth support ✅
- **Chrome on desktop** (Linux, Windows, macOS) — works ✅
- **iOS** — Safari doesn't support Web Bluetooth. Use [Bluefy](https://apps.apple.com/app/bluefy-web-ble-browser/id1492822055) or WebBLE browser
- **Firefox** — no Web Bluetooth support

Web Bluetooth requires a **secure context** (HTTPS or localhost).

## Quick Start

### Option 1: Local file server + Cloudflare tunnel (no USB cable)

```bash
# Serve the file
python3 -m http.server 8080 &

# Create a temporary HTTPS tunnel
cloudflared tunnel --url http://localhost:8080

# Open the printed https://xxx.trycloudflare.com/solarflow-bt.html on your phone
```

Install cloudflared on Ubuntu:
```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
```

### Option 2: Chrome USB port forwarding (Android + USB cable)

1. Enable USB debugging on your phone (Settings → Developer Options)
2. Plug phone into laptop via USB
3. On laptop: `python3 -m http.server 8080`
4. In Chrome on laptop: `chrome://inspect#devices` → Port Forwarding → `8080` → `localhost:8080`
5. On phone Chrome: open `localhost:8080/solarflow-bt.html`

## BLE Protocol

### GATT Service & Characteristics

All Zendure Solarflow devices expose a single BLE GATT service with two characteristics:

```
SERVICE:          0000a002-0000-1000-8000-00805f9b34fb
WRITE (command):  0000c304-0000-1000-8000-00805f9b34fb  (Handle: 0x002a)
NOTIFY (receive): 0000c305-0000-1000-8000-00805f9b34fb  (Handle: 0x002c)
```

All communication is JSON over these characteristics. Incoming notifications may be fragmented across multiple BLE packets and need to be reassembled by matching `{` / `}` braces.

### Connection & Handshake

After connecting, the device sends a BLESPP handshake:

```
← {"method":"BLESPP","deviceId":"<DEVICE_ID>"}
→ {"messageId":"1009","method":"BLESPP_OK"}
```

### Reading Device Info

```json
→ {"messageId":"1","method":"getInfo","timestamp":1234567890}
← {"messageId":"123","method":"getInfo-rsp","deviceId":"...","sn":"...","modules":[
     {"module":"MASTER","version":4372},
     {"module":"AC","version":4381},
     {"module":"BMS_AB2000","version":4117},
     {"module":"MPPT","version":4361},
     {"module":"ESP32","version":4362}
   ]}
```

### Reading All Properties

```json
→ {"messageId":"1","deviceId":"<ID>","timestamp":123,"properties":["getAll"],"method":"read"}
← {"method":"report","deviceId":"...","properties":{...}}   // multiple messages follow
← {"method":"report","deviceId":"...","packData":[...]}     // battery pack data
```

The device streams telemetry reports every ~5 seconds while the BLE connection is active.

### Writing Properties

Used for changing settings like output limit, SoC bounds, etc:

```json
→ {"method":"write","timestamp":123,"messageId":"abc","deviceId":"<ID>","properties":{"outputLimit":300}}
← {"method":"report","success":1,"deviceId":"...","properties":{"outputLimit":300}}
```

A successful write echoes the property back with `success:1`. If `properties` is empty `{}`, the device didn't recognize the property name.

### WiFi Provisioning

The WiFi provisioning protocol was reverse-engineered from Wireshark captures of the Zendure app's BLE traffic (Android HCI snoop log). It consists of two steps: scanning for networks, then sending credentials.

#### Step 1: WiFi Scan

The app asks the device to scan for visible WiFi networks:

```json
→ {"messageId":1001,"method":"WiFi.get"}
```

The device responds with one or more `WiFi.set` messages containing the scan results, terminated by an empty list:

```json
← {"messageId":1001,"method":"WiFi.set","WiFi.list":[
     {"SSID":"MyNetwork","signal":-33,"Primary":1,"AM":7},
     {"SSID":"FRITZ!Box 7690","signal":-43,"Primary":1,"AM":3},
     {"SSID":"Neighbor-5G","signal":-69,"Primary":1,"AM":7}
   ]}
← {"messageId":1001,"method":"WiFi.set","WiFi.list":[
     {"SSID":"OtherNetwork","signal":-89,"Primary":6,"AM":3}
   ]}
← {"messageId":1001,"method":"WiFi.set","WiFi.list":[]}
```

**WiFi.list fields:**

| Field | Type | Description |
|---|---|---|
| `SSID` | string | Network name |
| `signal` | int (dBm) | Signal strength (negative, higher = better) |
| `Primary` | int | WiFi channel |
| `AM` | int | Authentication mode — **critical for provisioning** |

**Known AM values:**

| AM | Likely Meaning |
|---|---|
| 3 | WPA / WPA2 mixed |
| 4 | WPA2-Enterprise or WPA3-transition |
| 7 | WPA2-PSK |

#### Step 2: Send Credentials (token method)

The app uses `Write Request` (opcode 0x12, with response) to send the credentials:

```json
→ {
    "AM": 7,
    "homeId": -67767571,
    "iotUrl": "mqtteu.zen-iot.com",
    "messageId": 1002,
    "method": "token",
    "password": "<wifi_password>",
    "ssid": "<wifi_ssid>",
    "timeZone": "GMT+01:00",
    "token": "<random_16_char_string>"
  }
← {"messageId":1002,"productModel":"R3mn8U","method":"Token","result":0}
```

The device then disconnects BLE and reboots to join the WiFi network.

**Token command fields:**

| Field | Type | Description |
|---|---|---|
| `AM` | int | WiFi authentication mode — **must match the value from the WiFi scan**. Without this field, or with the wrong value, the device returns `result:1` (error) and fails to connect. |
| `homeId` | int | Zendure cloud account/home ID. Negative integer. Can use a dummy value. |
| `iotUrl` | string | MQTT broker hostname. See table below. For local MQTT, use your broker's IP address. |
| `messageId` | int | Convention: `1002` |
| `method` | string | Must be `"token"` |
| `password` | string | WiFi password |
| `ssid` | string | WiFi SSID |
| `timeZone` | string | Timezone, e.g. `"GMT+01:00"` |
| `token` | string | 16-character random string. The app generates a new one per session. |

**Result codes:**

| result | Meaning |
|---|---|
| `0` | **Success** — device will reboot and connect to WiFi |
| `1` | **Error** — wrong AM, bad credentials, or missing fields |

> **Important**: Many community tools (solarflow-bt-manager, Zendure-HA) incorrectly interpret `result:1` as success. It is actually an error. The `AM` field from the WiFi scan is required for successful provisioning.

#### The `station` command

Some older tools send a follow-up command after the token:

```json
→ {"messageId":"1003","method":"station"}
```

Wireshark captures of the actual Zendure app show it does **not** send this command for the SF800 Pro. The device reboots immediately after accepting the token (`result:0`). The station command may be relevant for older Hub 1200/2000 firmware.

### MQTT Cloud Broker Hostnames

| Region | Hostname | Port |
|---|---|---|
| EU | `mqtteu.zen-iot.com` | 1883 |
| Global | `mq.zen-iot.com` | 1883 |

> **Note**: Some community documentation references `mqtt-eu.zen-iot.com` (with hyphen). Wireshark captures of the app show the actual hostname is `mqtteu.zen-iot.com` (no hyphen).

### BLE Write Modes

The Zendure app uses **Write Request** (ATT opcode `0x12`, expects a response) for WiFi provisioning commands. Some community tools use Write Command (opcode `0x52`, no response). Both may work for property writes, but WiFi provisioning is more reliable with Write Request.

## SF800 Pro Telemetry Properties

Properties observed from a SolarFlow 800 Pro (product key `R3mn8U`, firmware MASTER 4372, ESP32 4362):

### Power & Energy

| Property | Type | Description |
|---|---|---|
| `solarInputPower` | int (W) | Total solar input |
| `solarPower1` .. `solarPower4` | int (W) | Per-panel solar input |
| `outputHomePower` | int (W) | Power output to home/inverter |
| `outputPackPower` | int (W) | Power charging into batteries |
| `packInputPower` | int (W) | Power discharging from batteries |
| `gridInputPower` | int (W) | Grid input power |
| `gridOffPower` | int (W) | Grid-off power |

### Battery

| Property | Type | Description |
|---|---|---|
| `electricLevel` | int (%) | Overall battery percentage |
| `packState` | int | 0=idle, 1=charging, 2=discharging |
| `packNum` | int | Number of battery packs |
| `remainOutTime` | int (min) | Estimated discharge time (59940 = not discharging) |
| `BatVolt` | int | Battery voltage (raw) |

### Settings (readable and writable)

| Property | Type | Description |
|---|---|---|
| `outputLimit` | int (W) | Output limit to home |
| `inputLimit` | int (W) | Input limit |
| `minSoc` | int (‰) | Min SoC, 10x value (e.g. 50 = 5%) |
| `socSet` | int (‰) | Max SoC, 10x value (e.g. 1000 = 100%) |
| `inverseMaxPower` | int (W) | Inverter max power setting |
| `chargeMaxLimit` | int (W) | Maximum charge rate |
| `passMode` | int | Bypass: 0=auto, 1=off, 2=on |
| `pass` | int | Current bypass state |
| `buzzerSwitch` | int | Buzzer: 0=off, 1=on |
| `lampSwitch` | int | LED lamp: 0=off, 1=on |
| `acMode` | int | AC mode setting |
| `smartMode` | int | Smart mode setting |
| `gridReverse` | int | Grid reverse setting |
| `gridStandard` | int | Grid standard |
| `gridOffMode` | int | Grid-off mode |

### Status (read-only)

| Property | Type | Description |
|---|---|---|
| `IOTState` | int | IoT module state (1=active) |
| `bindstate` | int | Bound to account (0=no, 1=yes) |
| `heatState` | int | Heater state |
| `hyperTmp` | int | Device temperature (deci-Kelvin: `(val-2731)/10` = °C) |
| `pvStatus` | int | PV status |
| `acStatus` | int | AC status |
| `dcStatus` | int | DC status |
| `gridState` | int | Grid connection state |
| `dataReady` | int | Data ready flag |
| `Fanmode` | int | Fan mode |
| `Fanspeed` | int | Fan speed |
| `SN` | string | Device serial number |
| `productKey` | string | Product identifier |
| `deviceKey` | string | Device ID |
| `MASTER` | int | Master firmware version |
| `socLimit` | int | SoC limit status |
| `socStatus` | int | SoC status |
| `reverseState` | int | Reverse state |
| `writeRsp` | int | Write response status |
| `OTAState` | int | OTA update state |
| `LCNState` | int | LCN state |
| `factoryModeState` | int | Factory mode state |

### Battery Pack Data (per pack)

Reported in `packData` arrays within `report` messages:

| Property | Type | Description |
|---|---|---|
| `sn` | string | Pack serial number |
| `packType` | int | Pack type (300 = AB2000) |
| `socLevel` | int (%) | Pack charge level |
| `state` | int | 0=idle, 1=charging, 2=discharging |
| `power` | int (W) | Pack power |
| `maxTemp` | int | Max temp (deci-Kelvin: `(val-2731)/10` = °C) |
| `totalVol` | int | Total voltage (raw) |
| `batcur` | int | Battery current (raw, unsigned 16-bit) |
| `maxVol` | int | Max cell voltage |
| `minVol` | int | Min cell voltage |
| `softVersion` | int | Pack firmware version |
| `soh` | int (%) | State of health |

## Using with Home Assistant (SF800 Pro)

The SF800 Pro supports the **zenSDK** — a local HTTP API running on the device itself. Once WiFi is provisioned:

1. Find the device's IP in your router's DHCP leases
2. Give it a static IP lease
3. Block its internet access (router firewall rule)
4. Control it via HTTP from Home Assistant

See the [community guide for fully local SF800 Pro control](https://community.home-assistant.io/t/zendure-solarflow-800-pro-completely-local-zereo-feed-in-without-cloud-and-even-faster-no-mqtt-no-hacs-easy-mode/980110) for a complete HA package with zero feed-in, seasonal logic, emergency charging, and BMS calibration — no MQTT, no HACS, no cloud required.

> **Note**: The SF800 Pro does **not** support the legacy MQTT redirect approach used by older Hubs. Use the zenSDK HTTP API instead.

## Isolating the Device (OpenWrt)

```bash
# Static DHCP lease
uci add dhcp host
uci set dhcp.@host[-1].mac='XX:XX:XX:XX:XX:XX'
uci set dhcp.@host[-1].ip='192.168.1.200'
uci set dhcp.@host[-1].name='solarflow'
uci commit dhcp
/etc/init.d/dnsmasq restart

# Block internet access, keep LAN access
uci add firewall rule
uci set firewall.@rule[-1].name='Block Solarflow Internet'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].src_ip='192.168.1.200'
uci set firewall.@rule[-1].target='REJECT'
uci set firewall.@rule[-1].proto='all'
uci commit firewall
/etc/init.d/firewall restart
```

## Debugging BLE Traffic

### Capturing with Android HCI snoop log

1. Enable Developer Options (tap Build Number 7 times)
2. Settings → Developer Options → enable **"Enable Bluetooth HCI snoop log"**
3. **Toggle Bluetooth off then on** (critical — starts the log file)
4. Perform the BLE operation (WiFi provisioning, etc.)
5. Extract: `adb bugreport capture.zip`
6. Find `btsnoop_hci.log` in the zip:
   ```bash
   unzip capture.zip -d capture
   find capture/ -name "*btsnoop*"
   ```

### Analyzing with tshark

Extract all BLE writes and notifications with decoded JSON:

```bash
tshark -r btsnoop_hci.log \
  -Y "btatt.opcode == 0x12 || btatt.opcode == 0x52 || btatt.opcode == 0x1b" \
  -T fields -e frame.number -e frame.time_relative -e btatt.opcode -e btatt.handle -e btatt.value \
  2>/dev/null | while IFS=$'\t' read num time op handle val; do
    ascii=$(echo "$val" | xxd -r -p 2>/dev/null | tr -cd '[:print:]')
    echo "[$num] t=$time op=$op handle=$handle"
    echo "  $ascii"
    echo
done
```

**ATT opcodes:**
- `0x12` — Write Request (with response, used by the Zendure app)
- `0x52` — Write Command (without response, used by some community tools)
- `0x1b` — Handle Value Notification (device → phone)

### Wireshark filters

```
btatt.value contains "token"     # Find WiFi provisioning
btatt.value contains "method"    # Find all JSON commands
btatt.value contains "WiFi"      # Find WiFi scan
btatt                            # All ATT traffic
```

## Key Findings from Reverse Engineering

1. **The `AM` field is required** for WiFi provisioning on the SF800 Pro. Without it (or with the wrong value), the device returns `result:1` (error). The correct AM value comes from the device's own WiFi scan (`WiFi.get`).

2. **`result:0` = success, `result:1` = error.** Multiple community tools had this inverted, leading to false "success" reports while WiFi provisioning silently failed.

3. **The EU cloud MQTT hostname is `mqtteu.zen-iot.com`** (no hyphen), not `mqtt-eu.zen-iot.com` as documented elsewhere.

4. **The `station` command is not sent by the Zendure app** for SF800 Pro provisioning. The device reboots immediately after a successful token command.

5. **The Zendure app uses Write Request** (with response, ATT opcode 0x12), not Write Command (without response, 0x52). This matters because the device disconnects very quickly after accepting credentials.

6. **SF800 Pro uses zenSDK** (local HTTP API) instead of the MQTT redirect approach used by legacy Hubs. Once on WiFi, control it via HTTP at its IP address.

## Credits

- [reinhard-brandstaedter/solarflow-bt-manager](https://github.com/reinhard-brandstaedter/solarflow-bt-manager) — original Python BLE manager, protocol reverse engineering
- [epicRE/zendure_ble](https://github.com/epicRE/zendure_ble) — BLE protocol documentation for Hub 1200
- [Zendure/Zendure-HA](https://github.com/Zendure/Zendure-HA) — official Home Assistant integration (BLE provisioning reference)
- [nograx/zendure-cloud-disconnector](https://github.com/nograx/zendure-cloud-disconnector) — Windows BLE disconnector tool
- WiFi provisioning protocol details captured via Android HCI snoop log + Wireshark analysis of the Zendure app

## License

MIT
