# Solarflow Web Bluetooth Manager

A browser-based tool to manage Zendure Solarflow devices over Bluetooth Low Energy (BLE) — no app, no Python, no Raspberry Pi required. Runs directly from your phone or laptop using the Web Bluetooth API.

Built as a single HTML file with zero dependencies (React loaded from CDN).

## Features

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

### Connection

All Zendure Solarflow devices expose a single BLE GATT service with two characteristics:

```
SERVICE:          0000a002-0000-1000-8000-00805f9b34fb
WRITE (command):  0000c304-0000-1000-8000-00805f9b34fb
NOTIFY (receive): 0000c305-0000-1000-8000-00805f9b34fb
```

After connecting, the device sends a handshake:

```
← {"method":"BLESPP","deviceId":"<DEVICE_ID>"}
→ {"messageId":"1009","method":"BLESPP_OK"}
```

### Reading Device Info

```json
→ {"messageId":"1","method":"getInfo","timestamp":1234567890}
← {"messageId":"123","method":"getInfo-rsp","deviceId":"...","deviceSn":"...","modules":[...]}
```

### Reading All Properties

```json
→ {"messageId":"1","deviceId":"<ID>","timestamp":123,"properties":["getAll"],"method":"read"}
← {"method":"report","deviceId":"...","properties":{...}}   // multiple messages
← {"method":"report","deviceId":"...","packData":[...]}     // battery pack data
```

### Writing Properties

Used for changing settings like output limit, SoC bounds, etc:

```json
→ {"method":"write","timestamp":123,"messageId":"abc","deviceId":"<ID>","properties":{"outputLimit":300}}
← {"method":"report","success":1,"deviceId":"...","properties":{"outputLimit":300}}
```

Note: a successful write echoes the property back. If `properties` is empty `{}`, the device didn't recognize the property name.

### WiFi Provisioning (token method)

This is the critical command for connecting the device to WiFi. The exact format was reverse-engineered from Wireshark captures of the Zendure app.

**Important**: the Zendure app uses `Write Request` (opcode 0x12, with response), not `Write Command` (opcode 0x52, without response). Some older documentation incorrectly suggests write-without-response.

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
    "token": "<16_char_string>"
  }
← {"messageId":30,"productModel":"R3mn8U","method":"Token","result":1}
  (device disconnects BLE and reboots to join WiFi)
```

**Fields:**

| Field | Type | Description |
|---|---|---|
| `AM` | int | WiFi authentication mode. `7` = WPA2. **Required for SF800 Pro.** |
| `homeId` | int | Zendure cloud account/home ID. Negative integer. |
| `iotUrl` | string | MQTT broker hostname. `mqtteu.zen-iot.com` for EU cloud, `mq.zen-iot.com` for global. For local MQTT, use your broker's IP. Note: EU hostname is `mqtteu` (no hyphen), not `mqtt-eu`. |
| `messageId` | int | `1002` (convention from the app) |
| `method` | string | Must be `"token"` |
| `password` | string | WiFi password |
| `ssid` | string | WiFi SSID |
| `timeZone` | string | Timezone string, e.g. `"GMT+01:00"` |
| `token` | string | 16-character token string. The app generates a random one per session. |

**The `station` command:**

Some older tools (solarflow-bt-manager, Zendure-HA integration) send a follow-up command:

```json
→ {"messageId":"1003","method":"station"}
```

However, Wireshark captures of the actual Zendure app show it does **not** send this command. The device reboots immediately after accepting the token. The station command may be needed for older Hub 1200/2000 firmware but is not required (and cannot be delivered in time) for the SF800 Pro.

### MQTT Cloud Broker Hostnames

| Region | Hostname | Port |
|---|---|---|
| EU | `mqtteu.zen-iot.com` | 1883 |
| Global | `mq.zen-iot.com` | 1883 |

Note: some community documentation references `mqtt-eu.zen-iot.com` (with hyphen). The actual app uses `mqtteu.zen-iot.com` (no hyphen).

## SF800 Pro Telemetry Properties

Properties observed from a SolarFlow 800 Pro (product key `R3mn8U`, firmware MASTER 4372):

### Power & Energy

| Property | Type | Description |
|---|---|---|
| `solarInputPower` | int (W) | Total solar input |
| `solarPower1` | int (W) | Panel 1 input |
| `solarPower2` | int (W) | Panel 2 input |
| `solarPower3` | int (W) | Panel 3 input |
| `solarPower4` | int (W) | Panel 4 input |
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
| `remainOutTime` | int (min) | Estimated discharge time remaining (59940 = not discharging) |
| `BatVolt` | int | Battery voltage (raw, divide by 100 for volts) |

### Settings

| Property | Type | Description |
|---|---|---|
| `outputLimit` | int (W) | Output limit to home |
| `inputLimit` | int (W) | Input limit |
| `minSoc` | int (‰) | Min SoC, 10x (e.g. 50 = 5%) |
| `socSet` | int (‰) | Max SoC, 10x (e.g. 1000 = 100%) |
| `inverseMaxPower` | int (W) | Inverter max power setting |
| `passMode` | int | Bypass mode: 0=auto, 1=off, 2=on |
| `pass` | int | Current bypass state |
| `buzzerSwitch` / `lampSwitch` | int | Buzzer/lamp on(1) or off(0) |
| `chargeMaxLimit` | int (W) | Maximum charge rate |
| `acMode` | int | AC mode setting |
| `smartMode` | int | Smart mode setting |
| `gridReverse` | int | Grid reverse setting |
| `gridStandard` | int | Grid standard setting |
| `gridOffMode` | int | Grid-off mode |

### Status

| Property | Type | Description |
|---|---|---|
| `IOTState` | int | IoT module state (1=active) |
| `bindstate` | int | Whether device is bound to an account (0=no) |
| `heatState` | int | Heater state |
| `hyperTmp` | int | Device temperature (deci-Kelvin, convert: (val-2731)/10 = °C) |
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

### Battery Pack Data (per pack)

| Property | Type | Description |
|---|---|---|
| `sn` | string | Pack serial number |
| `packType` | int | Pack type (300 = AB2000) |
| `socLevel` | int (%) | Pack charge level |
| `state` | int | 0=idle, 1=charging, 2=discharging |
| `power` | int (W) | Pack power |
| `maxTemp` | int | Max temp (deci-Kelvin: (val-2731)/10 = °C) |
| `totalVol` | int | Total voltage (raw) |
| `batcur` | int | Battery current (raw, unsigned) |
| `maxVol` | int | Max cell voltage |
| `minVol` | int | Min cell voltage |
| `softVersion` | int | Pack firmware version |
| `soh` | int (%) | State of health |

## Using with Home Assistant (SF800 Pro)

The SF800 Pro supports the **zenSDK** — a local HTTP API on the device. Once WiFi is provisioned:

1. Find the device's IP in your router's DHCP leases
2. Give it a static IP lease
3. Block its internet access (router firewall rule)
4. Control it via HTTP from Home Assistant

See the [community guide for fully local SF800 Pro control](https://community.home-assistant.io/t/zendure-solarflow-800-pro-completely-local-zereo-feed-in-without-cloud-and-even-faster-no-mqtt-no-hacs-easy-mode/980110) for a complete HA package with zero feed-in, seasonal logic, emergency charging, and BMS calibration.

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
2. Enable "Bluetooth HCI snoop log"
3. Toggle Bluetooth off/on to start recording
4. Perform the BLE operation
5. Extract: `adb bugreport capture.zip`
6. Find `btsnoop_hci.log` in the zip

### Analyzing with tshark

```bash
# All ATT writes and notifications with decoded JSON
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

## Credits

- [reinhard-brandstaedter/solarflow-bt-manager](https://github.com/reinhard-brandstaedter/solarflow-bt-manager) — original Python BLE manager, protocol reverse engineering
- [epicRE/zendure_ble](https://github.com/epicRE/zendure_ble) — BLE protocol documentation
- [Zendure/Zendure-HA](https://github.com/Zendure/Zendure-HA) — official Home Assistant integration (BLE provisioning reference)
- [nograx/zendure-cloud-disconnector](https://github.com/nograx/zendure-cloud-disconnector) — Windows BLE disconnector tool

## License

MIT
