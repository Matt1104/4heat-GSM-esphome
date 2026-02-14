# 4Heat GSM Simulator for ESPHome

Custom ESPHome component to simulate a 4Heat Tiemme GSM/WiFi module, enabling remote control of pellet stoves/boilers via Home Assistant.

## ⚡ What's New in v2.0

- ✅ **ESP32 support with ESP-IDF framework** (required by ESPHome 2026+)
- ✅ **GSM/SMS protocol simulation** for command sending via UART
- ✅ **MB250 compatibility** (Tieffe pellet stove controller)
- ✅ **RS232 communication** via MAX3232
- ✅ Backward compatibility with ESP8266 (Arduino framework)

## 🔧 Supported Hardware

### Microcontrollers
- **ESP32-WROOM-32D** (recommended, ESP-IDF framework)
- **ESP32-C3** (ESP-IDF framework)
- **ESP8266 D1 Mini** (legacy, Arduino framework)

### Required Hardware
- **RS232-TTL converter:** MAX3232 or equivalent
- **Straight DB9 cable** (not null-modem)
- 5V USB or external power supply

## 📡 Wiring Diagram

### ESP32 + MAX3232 + MB250

```
ESP32-WROOM-32D          MAX3232                MB250 (DB9 Female)
┌──────────────┐      ┌──────────┐           ┌─────────────┐
│              │      │          │           │             │
│  GPIO17 (TX) ├─────►│ T1IN(11) │           │             │
│  GPIO16 (RX) │◄─────┤ R1OUT(12)│           │             │
│              │      │          │           │             │
│  5V (VIN)    ├─────►│ VCC (16) │           │             │
│  GND         ├──┬──►│ GND (15) │           │             │
│              │  │   │          │           │             │
└──────────────┘  │   │ T1OUT(14)├──────────►│ Pin 2 (RX)  │
                  │   │ R1IN(13) │◄──────────┤ Pin 3 (TX)  │
                  │   │          │           │ Pin 5 (GND) │◄─┘
                  │   └──────────┘           └─────────────┘
                  │
                  └─────────────────────────────────────────┘
```

**Notes:**
- MAX3232 VCC: 3.3V-5.5V (5V preferred)
- MAX3232 capacitors: 4-5x 0.1µF as per datasheet
- DB9 pins 2/3 might be swapped on some MB250 models

## 📦 Installation

### 1. Add custom component to ESPHome

```yaml
external_components:
  - source: github://matt1104/4heat-GSM-esphome
    components: [fourheat]
    refresh: 0s
```

### 2. Basic Configuration

```yaml
esphome:
  name: camino-mb250
  friendly_name: "4Heat GSM Simulator"

esp32:
  board: esp32dev
  framework:
    type: esp-idf  # REQUIRED for ESPHome 2026+

logger:
  level: DEBUG
  baud_rate: 0  # Disable logging on UART0 to avoid conflicts

api:
ota:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

uart:
  id: mb250_uart
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 9600

fourheat:
  id: fourheat_main
  uart_id: mb250_uart
  update_interval: 20s
```

### 3. Configure Sensors and Buttons

```yaml
sensor:
  - platform: fourheat
    name: "Water Temperature"
    datapoint: J30017
    device_class: temperature
    unit_of_measurement: "°C"

  - platform: fourheat
    name: "Flue Gas Temperature"
    datapoint: J30005
    device_class: temperature
    unit_of_measurement: "°C"

  - platform: fourheat
    name: "Stove Status Raw"
    id: stato_raw
    datapoint: J30001

button:
  - platform: template
    name: "Turn On Stove"
    icon: "mdi:fire"
    on_press:
      - lambda: |-
          id(fourheat_main)->send_gsm_command("Start");

  - platform: template
    name: "Turn Off Stove"
    icon: "mdi:fire-off"
    on_press:
      - lambda: |-
          id(fourheat_main)->send_gsm_command("Stop");

text_sensor:
  - platform: template
    name: "Operating Status"
    lambda: |-
      int mode = id(stato_raw).state;
      switch(mode) {
        case 0: return {"OFF"};
        case 1: return {"Check Up"};
        case 2: return {"Ignition"};
        case 3: return {"Stabilization"};
        case 5: return {"Running"};
        case 6: return {"Modulation"};
        case 7: return {"Shutdown"};
        case 8: return {"Safety"};
        case 9: return {"Blocked"};
        case 11: return {"Standby"};
        default: return {"Status " + to_string(mode)};
      }
```

## 🎮 Simulated GSM Commands

The component simulates SMS sending to the MB250 controller:

| Lambda Command | Simulated SMS | Action |
|----------------|---------------|--------|
| `send_gsm_command("Start")` | "Start" | Turn on stove |
| `send_gsm_command("Stop")` | "Stop" | Turn off stove |
| `send_gsm_command("Reset")` | "Reset" | Reset alarms |

### Simulated GSM Protocol Flow

```
1. ESP32 sends:  +CMTI: "SM",1     (new SMS notification)
2. MB250 asks:   AT+CMGR=1         (read SMS)
3. ESP32 replies: +CMGR: "REC UNREAD","+39xxx"
                  Start             (SMS content)
                  OK
4. MB250 executes command
5. MB250 sends:  AT+CMGS="+39xxx"  (SMS confirmation)
6. ESP32 replies: OK
```

## 📊 Available Datapoints

| ID | Description | Unit |
|----|-------------|------|
| J30001 | Operating status | - |
| J30002 | Error code | - |
| J30005 | Flue gas temperature | °C |
| J30011 | Combustion power | - |
| J30012 | Buffer temperature | °C |
| J30017 | Water temperature | °C |
| J30020 | Water pressure | mbar |

## 🐛 Troubleshooting

### `uart driver error` on startup
**Solution:** Verify that the framework is set to `esp-idf` in YAML

### Corrupted data or `incomplete data`
**Possible causes:**
1. **TX/RX swapped:** Swap GPIO17 ↔ GPIO16 in YAML
2. **No GND connection:** Check GND continuity between ESP32, MAX3232 and MB250
3. **Null-modem cable:** Use straight DB9 cable (not crossed)
4. **MAX3232 underpowered:** Verify 5V on pin 16

### Sensors not updating
**Check:**
- ESPHome logs: look for `Received data for J30xxx`
- Controller is powered on and connected
- Correct baud rate (9600)

### Testing without MB250 hardware

You can test the component without physical connection:
- Disconnect RX pin (GPIO16) temporarily
- Use buttons to send GSM commands
- Check logs for `>>> SIMULAZIONE SMS: Start <<<`

## 📝 Complete Configuration Example

See [`4heat.yaml`](4heat.yaml) file in the repository for a full working configuration.

## 🔧 Development

### Building from source

```bash
git clone https://github.com/Matt1104/4heat-GSM-esphome.git
cd 4heat-GSM-esphome

# Test compilation
esphome compile 4heat.yaml
```

### File structure

```
4heat-GSM-esphome/
├── fourheat.h           # Component header
├── fourheat.cpp         # Component implementation
├── __init__.py          # Python configuration schema
├── sensor/              # Sensor platform
├── 4heat.yaml           # Example configuration
└── README.md
```

## 🤝 Credits

- **Original project:** [leoshusar/4heat-esphome](https://github.com/leoshusar/4heat-esphome)
- **GSM simulator fork:** Matt1104
- **Framework:** [ESPHome](https://esphome.io)

## 📜 License

MIT License - see LICENSE file

## ⚠️ Disclaimer

This project is provided "as is" without warranties. Improper use could damage the controller hardware. Always test on non-critical hardware before final installation.

---

**Compatible with ESPHome 2026.1.5+ and Home Assistant 2026+**

## 🌟 Support

If you find this project useful, please give it a ⭐ on GitHub!

For issues and feature requests, please use the [GitHub Issues](https://github.com/Matt1104/4heat-GSM-esphome/issues) page.
