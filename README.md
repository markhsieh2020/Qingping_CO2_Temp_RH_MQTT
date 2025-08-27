# Qingping_CO2_Temp_RH_MQTT

**BLE-to-MQTT bridge for Qingping CO₂ Temp RH sensors using ESP32-C3, with full Home Assistant integration.**

📌 For **Qingping Temp & RH Barometer Pro (CGP23W)** setup, please refer to my other repository:  
👉 [Qingping_Barometer_Pro_CGP23W_MQTT](https://github.com/markhsieh2020/Qingping_Barometer_Pro_CGP23W_MQTT)  
---

## 🚀 Project Overview
This project connects a **Qingping CO₂ Temp RH (CGP22C)** sensor (via BLE) to **Home Assistant** using an **ESP32-C3 PRO MINI (ESP32-C3FH4)** as a bridge.  
Sensor data is published over **MQTT with Autodiscovery**, enabling seamless integration with Home Assistant.  

- The **Qingping CO₂ Temp RH (CGP22C)** broadcasts BLE data roughly **once per second**.  
- In this example, the ESP32-C3 is configured to **publish to MQTT every 5 seconds**.  
- ⚙️ This **5-second interval can be modified** in the source code to suit your requirements.
  
**PS:** According to the [official Qingping documentation](https://developer.qingping.co/), you can configure **private MQTT mode**.  
However, based on testing:  
- Once private MQTT is enabled, the **Qingping+ app** can no longer display data or modify settings.  
- You will need to **reset the Qingping CO₂ Temp RH (CGP22C)** to restore app functionality.  
- The official Qingping+ app supports **offline buffering**, temporarily storing data when offline and uploading it once the device reconnects.  
- ⚡ If you want to **keep this offline history feature**, use this **BLE-to-MQTT bridge solution** instead of switching to private MQTT mode.  
---

## ✨ Features
- **BLE Scanning**: Reads Qingping BLE advertisements (UUID `0xFDCD`).  
- **Sensor Data**: Publishes CO₂ (ppm), Temperature (°C), Humidity (%), Battery (%), BLE RSSI (dBm).  
- **Diagnostics**: Reports Boot Count, LAN/WAN IP, MAC address, MQTT error count, Reset Reason, Wi-Fi BSSID, Channel, Reconnects, Wi-Fi RSSI Proxy, SSID, and Uptime.  
- **Controls**:  
  - Restart Button (via MQTT in Home Assistant)  
  - On-board LED Switch (blinks 3x on boot, defaults OFF)  
- **MQTT Autodiscovery**: Auto-creates sensors and controls in Home Assistant.  
- **Reliability**: Auto-reconnect to Wi-Fi & MQTT, WAN IP refresh, persistent boot counter in NVS.  

---

## 🛠 Hardware Requirements
- **Board**: ESP32-C3 PRO MINI (ESP32-C3FH4, 4 MB Flash, Wi-Fi + BLE 5)  
- **Sensor**: Qingping CO₂ Temp RH (**CGP22C**)  
- **On-board LED**: GPIO8 (active LOW, blinks 3x at boot, default OFF)  

---

## 📦 Software Requirements
- **Arduino IDE** ≥ **2.3.6**  
- **ESP32 Arduino Core** ≥ **2.0**  
- **Required Libraries (install via Arduino Library Manager):**  
  1. **NimBLE-Arduino** → version **2.3.4**  
     - *Library Manager*: search for `NimBLE-Arduino` by **h2zero**  
  2. **PubSubClient** → version **2.8**  
     - *Library Manager*: search for `PubSubClient` by **Nick O’Leary**  
  3. **ArduinoJson** → version **7.4.2**  
     - *Library Manager*: search for `ArduinoJson` by **Benoît Blanchon**  

---

## ⚡ Installation
1. Open this file in your browser:  
   👉 [esp32c3_qingping_co2_mqtt.txt](https://github.com/markhsieh2020/Qingping_CO2_Temp_RH_MQTT/blob/main/esp32c3_qingping_co2_mqtt.txt)  
2. Copy all the code inside the `.txt` file.  
3. Install required libraries (see above).  
4. Edit Wi-Fi and MQTT credentials in the source code:  
   ```cpp
   static const char* ssid        = "YOUR_WIFI_SSID";
   static const char* password    = "YOUR_WIFI_PASSWORD";
   static const char* mqtt_server = "YOUR_MQTT_SERVER";
   static const int   mqtt_port   = 1883;
   static const char* mqtt_user   = "mqtt_user";
   static const char* mqtt_pass   = "mqtt_pass";
   ```
5. Replace the **BLE MAC address** with your own sensor’s:  
   ```cpp
   static const char* TARGET_BLE_ADDR = "58:2d:34:84:54:0c";
   ```
   ⚠️ **Important**: Your device has a unique MAC. Find it using the **Qingping+ iOS app** (App Store → device info).  
6. Compile and flash to ESP32-C3.  

---

## ⚙️ Configuration
- **MQTT Discovery Prefix**: `homeassistant`  
- **MQTT Topics Example**:  
  - State: `ESP32_C3_BLE_xxxxxx/state`  
  - Availability: `ESP32_C3_BLE_xxxxxx/availability`  
  - Commands:  
    - Restart → `ESP32_C3_BLE_xxxxxx/cmd/restart`  
    - LED → `ESP32_C3_BLE_xxxxxx/cmd/led`  

---

## 📡 BLE Advertisement Parsing

The Qingping CGP22C broadcasts sensor data via BLE (UUID `0xFDCD`). Example raw broadcast:  

```
== ADV == name="Qingping CO2 Temp RH" addr=58:2d:34:84:54:0c rssi=-48
FDCD SD : 88 33 0C 54 84 34 2D 58 01 04 23 01 D2 01 02 01 64 13 02 41 03
```

### Breakdown
- **Device ID**: `88 33 0C 54 84 34 2D 58` (ignored in parsing)  
- **Temperature + Humidity**: `01 04 23 01 D2 01` → 29.1 °C / 46.6 %  
- **Battery**: `02 01 64` → 100 %  
- **CO₂**: `13 02 41 03` → 833 ppm  
- **RSSI**: From scanner = –48 dBm  

### ✅ Final Decoded Values
- Temperature: **29.1 °C**  
- Humidity: **46.6 %**  
- Battery: **100 %**  
- CO₂: **833 ppm**  
- RSSI: **–48 dBm**  

### 💻 C++ Parsing Example
```cpp
bool parseFDCD_TLV(const std::vector<uint8_t>& sd, QingpingCO2Parsed& o) {
  if (sd.size() < 14) return false;
  size_t pos = 8; // skip 8-byte device ID

  auto rd16s=[&](size_t off)->int16_t {return (int16_t)(sd[off] | (sd[off+1] << 8));};
  auto rd16u=[&](size_t off)->uint16_t{return (uint16_t)(sd[off] | (sd[off+1] << 8));};

  pos += 2; // key,dtype
  int16_t  t_raw = rd16s(pos); pos += 2;
  uint16_t h_raw = rd16u(pos); pos += 2;
  o.temperature_c = t_raw / 10.0f;
  o.humidity_rh   = h_raw / 10.0f;

  o.battery_pct   = -1;
  o.co2_ppm       = -1;

  while (pos + 1 < sd.size()) {
    uint8_t tag = sd[pos++], len = sd[pos++];
    if (pos + len > sd.size()) break;
    if (tag == 0x02 && len == 1)      o.battery_pct = sd[pos];
    else if (tag == 0x13 && len == 2) o.co2_ppm     = rd16u(pos);
    pos += len;
  }
  return (o.co2_ppm >= 0);
}
```

---

## 🧪 Tested Environment

### Development
- **Arduino IDE**: 2.3.6  
- **ESP32 Arduino Core**: 2.0+  
- **Libraries**: NimBLE-Arduino v2.3.4, PubSubClient v2.8, ArduinoJson v7.4.2  

### Hardware
- **Board**: ESP32-C3 PRO MINI (ESP32-C3FH4, 4 MB Flash)  
- **Sensor**: Qingping CO₂ Temp RH (CGP22C)  
- **On-board LED**: GPIO8 (active LOW)  

### Home Assistant
```json
{
  "installation_type": "Home Assistant OS",
  "version": "2025.7.4",
  "python_version": "3.13.3",
  "docker": true,
  "arch": "x86_64",
  "timezone": "Asia/Taipei",
  "os_name": "Linux",
  "os_version": "6.12.35-haos",
  "container_arch": "amd64",
  "supervisor": "2025.08.1",
  "host_os": "Home Assistant OS 16.0",
  "docker_version": "28.3.0",
  "chassis": "vm",
  "run_as_root": true
}
```

### ⚠️ Compatibility Notes
- Tested on **ESP32-C3 PRO MINI (ESP32-C3FH4)** only.  
- Should also work on other ESP32-C3 boards (adjust LED pin if needed).  
- Requires **MQTT Autodiscovery enabled** in Home Assistant.  

---

## 📦 Device Information

- **Model**: Qingping CO₂ Temp RH (**CGP22C**)  
- **Firmware Version**: **2.0.6** (tested)  
- BLE MAC address must be replaced with your own device’s (find in **Qingping+ iOS app → Device Info**).  

---

## 📋 Known Working Versions

| Model                      | Firmware | Tested | Notes                          |
|-----------------------------|----------|--------|--------------------------------|
| Qingping CO₂ Temp RH CGP22C | 2.0.6    | ✅ Yes | BLE parsing verified in this repo |

---

## 🔧 Troubleshooting
- **No MQTT entities in HA?** → Ensure `homeassistant/` discovery topics are enabled.  
- **Wi-Fi disconnects?** → Ensure stable 2.4GHz Wi-Fi.  
- **Wrong sensor values?** → Confirm your sensor firmware and MAC address.  

---

## 📜 License
This project is licensed under the **MIT License**. See [LICENSE](LICENSE) for details.  

---

## 🙏 Acknowledgements
- [NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino)  
- [ArduinoJson](https://arduinojson.org/)  
- [PubSubClient](https://pubsubclient.knolleary.net/)  
