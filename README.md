# Sonoff TH10 ESPHome Custom Firmware & Flashing Guide

This repository contains the custom [ESPHome](https://esphome.io/) configuration and step-by-step hardware flashing guide for the **Sonoff TH10** (ESP8266-based temperature and humidity relay switch).

---

## Hardware Overview

The Sonoff TH10 is an ESP8266-based smart relay that includes a 2.5mm TRRS jack for connecting external temperature/humidity sensors (e.g., AM2301, Si7021, DS18B20).

| Exterior Case | Sensor Attachment |
| :---: | :---: |
| ![Sonoff TH10 Case](<Sonoff TH10 Case.jpg>) | ![Sonoff TH10 with Sensor](<Sonoff TH10 with Humidity and temp sensor.jpg>) |

| AM2301 / Si7021 Sensor |
| :---: |
| ![AM2301 Sensor](<Sonoff TH10 temp and humidity sensor AM2301.jpg>) |

---

## ⚠️ Critical Safety Warning

> **DANGER: HIGH VOLTAGE RISK**
> - **NEVER** connect the Sonoff TH10 to AC mains power (100–240V AC) while the case is open or while connecting to a USB-to-serial adapter.
> - When flashing, power the board **STRICTLY and ONLY** from the **3.3V** pin of your USB-to-UART adapter.
> - **DO NOT USE 5V** — the ESP8266 operates strictly at 3.3V logic and VCC. Applying 5V directly to the ESP8266 power rail will destroy the chip.

---

## PCB Pinout & Serial Header

Inside the Sonoff TH10 case, you will find a 4-pin header footprint on the PCB adjacent to the ESP8266 and the push button.

| PCB Front Overview | PCB Rear |
| :---: | :---: |
| ![PCB Front](<Sonoff TH10 pcb front.jpg>) | ![PCB Rear](<Sonoff TH10 pcb rear.jpg>) |

### Pin Definitions

| Sonoff Header Pin | ESP8266 Function | USB-Serial Adapter Connection | Notes |
| :--- | :--- | :--- | :--- |
| **VCC / 3V3** | Power (3.3V) | **3.3V** | Must be 3.3V; never 5V |
| **RX** | UART RX (GPIO3) | **TX** | Cross-over to Adapter TX |
| **TX** | UART TX (GPIO1) | **RX** | Cross-over to Adapter RX |
| **GND** | Ground | **GND** | Common ground reference |
| **GPIO14** *(if present on header)* | Sensor Data | *Leave Disconnected* | Connected to the 2.5mm sensor jack |

### Wiring Examples

| Serial Header Connection | Detailed View |
| :---: | :---: |
| ![Serial Connection 1](<Sonoff TH10 serial connection.jpg>) | ![Serial Connection 2](<Sonoff TH10 serial connection2.jpg>) |

| 3.3V Power Line Detail | USB-Serial Adapter Wiring |
| :---: | :---: |
| ![PCB 3V3 Connection](<Sonoff TH10 pcb serial connection 3v3.jpg>) | ![Serial Adapter Wiring](<Sonoff TH10 3v3 navspark board serial rx tx connection.jpg>) |

---

## ESP8266 GPIO Map

| GPIO | Component / Function | Configuration Notes |
| :--- | :--- | :--- |
| **GPIO0** | Physical Push Button | Active LOW (`INPUT_PULLUP`, `inverted: true`). Holding at boot enters flash mode. |
| **GPIO12** | Main Power Relay | Active HIGH (`RESTORE_DEFAULT_OFF`). |
| **GPIO13** | Blue Status LED | Active LOW (`inverted: true`). |
| **GPIO14** | Sensor Bus (2.5mm Jack) | DHT / AM2301 / Si7021 / DS18B20 data line. |

---

## Prerequisites

1. **USB-to-UART Adapter** (e.g., CP2102, FT232RL, CH340G) set to **3.3V logic level**.
2. **ESPHome CLI** installed (via `uv`, `pip`, or ESPHome Dashboard):
   ```bash
   # Using uvx (recommended, no install needed)
   uvx esphome --version

   # Or via pip
   pip install esphome
   ```
3. Dialout permissions on Linux:
   ```bash
   sudo usermod -a -G dialout $USER
   ```

---

## Flashing Instructions

### 1. Configure Secrets

Copy the example secrets file and edit with your Wi-Fi credentials and API encryption key:

```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml`:
```yaml
wifi_ssid: "MyHomeWiFi"
wifi_password: "MySecretPassword"
api_encryption_key: "BASE64_KEY_HERE"
```

*(You can generate a new base64 key with `openssl rand -base64 32` or via ESPHome).*

### 2. Enter Bootloader / Flash Mode

1. Ensure the Sonoff TH10 is **completely disconnected** from AC mains and from USB power.
2. Connect **GND**, **TX**, and **RX** between your USB-UART adapter and the Sonoff TH10 header.
3. **Press and hold down the physical push-button** on the Sonoff TH10 (GPIO0).
4. While continuing to hold the button, plug the USB adapter into your computer (or connect the **3.3V** power wire).
5. Keep holding the button for **2–3 seconds**, then release it.
6. The ESP8266 is now in flashing mode.

### 3. Compile and Flash

Run the compile and upload command:

```bash
# Compile and flash via serial port
uvx esphome run sonoffth10.yaml --device /dev/ttyUSB0
```

ESPHome will:
- Download the ESP8266 platform and libraries.
- Compile the C++ firmware image.
- Connect via `esptool` and flash the binary to the 1MB flash chip.

```text
Connecting....
Connected to ESP8266 on /dev/ttyUSB0
Flash will be erased from 0x00000000 to 0x00073fff...
Writing at 0x00073800 [==============================] 100.0%
Wrote 473088 bytes at 0x00000000 in 8.8 seconds.
Hash of data verified.
Hard resetting via RTS pin...
INFO Successfully uploaded program.
```

### 4. Boot into Normal Mode

1. Unplug the 3.3V power / USB adapter.
2. Reconnect the 3.3V power **without** touching the push-button.
3. The board will boot the new ESPHome firmware and connect to your Wi-Fi.
4. Future updates can be flashed Over-The-Air (OTA) directly without opening the case again:
   ```bash
   uvx esphome run sonoffth10.yaml
   ```

---

## Home Assistant Integration

Once powered and connected to Wi-Fi:
1. Home Assistant will auto-discover the Sonoff TH10 via native ESPHome API integration.
2. Click **Configure** and enter your `api_encryption_key` if prompted.
3. The following entities will be available:
   - **Switch:** Relay control (`switch.relay`)
   - **Binary Sensor:** Physical push button state
   - **Sensors:** Temperature, Humidity, Wi-Fi signal strength (RSSI), Controller Uptime
   - **Button:** Restart Controller
