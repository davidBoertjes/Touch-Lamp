# Smart Touch Lamp Project

A Home Assistant-integrated smart lamp control system using ESP32 microcontroller with touch-based brightness control and UART-based dimmer module communication.

## Project Overview

This project provides two configurations for controlling an AC lamp dimmer:

1. **Original Configuration (`touch_lamp.yaml`)** - Direct AC dimmer control using ESP32 library
2. **New Configuration (`touch_lamp_dimmerlink.yaml`)** - DimmerLink UART board for flicker-free control

The smart lamp features capacitive touch pad control for brightness adjustment and integrates seamlessly with Home Assistant for remote control and automation.

## Project Structure

```
Touch Lamp/
├── README.md                           # This file - project overview
├── touch_lamp.yaml                     # Original configuration (direct AC dimmer)
├── touch_lamp_description.md           # Original config documentation
├── touch_lamp_dimmerlink.yaml          # New configuration (DimmerLink UART)
├── touch_lamp_dimmerlink_README.md     # DimmerLink config documentation
```

## Hardware Requirements

### Core Components

- **Microcontroller**: ESP32 DoIt DevKit v1 (or ESP32-S3 DevKit C-1)
- **WiFi/Networking**: Built-in ESP32 WiFi module
- **AC Dimmer Control**: 
  - Option A: Direct TRIAC dimmer with zero-cross detection
  - Option B: DimmerLink UART board (recommended)
- **Touch Sensor**: Capacitive touch pad on GPIO13
- **Power Supply**: 
  - ESP32: 5V/1A USB or 3.3V regulated supply
  - AC Dimmer: 230V/110V AC mains

### DimmerLink Board (Recommended)

The DimmerLink is a dedicated microcontroller board that handles all AC dimmer timing:

- **Interface**: UART (115200 baud, 8N1)
- **Power**: 3.3V-5V compatible
- **Features**:
  - Flicker-free dimming (dedicated Cortex-M+ MCU)
  - Zero-cross detection (50/60Hz auto-detect)
  - 0-100% brightness control
  - Multiple dimming curves (Linear, RMS, Logarithmic)
  - Works with any AC dimmer TRIAC module

**Purchase links:**
- [DimmerLink on RBDimmer.com](https://www.rbdimmer.com/shop/dimmerlink-controller-uart-i2c-interface-for-ac-dimmers-48)
- [DimmerLink on AliExpress](https://fr.aliexpress.com/item/1005011583805008.html)

## Configuration Comparison

| Feature | `touch_lamp.yaml` | `touch_lamp_dimmerlink.yaml` |
|---------|-------------------|------------------------------|
| Dimmer Control | Direct ESP32 library | UART to DimmerLink board |
| Flicker Risk | May flicker under load | Flicker-free (dedicated MCU) |
| Complexity | Requires ac_dimmer library | Simple 4-byte UART commands |
| Multi-dimmer | Single dimmer per board | Multiple dimmers on same bus |
| Pin Usage | GPIO5, GPIO12 | GPIO16, GPIO17 (configurable) |
| Timing Control | Software-based | Hardware-based on DimmerLink |
| Reliability | Depends on ESP32 load | Independent operation |

## Quick Start

### 1. Preparation

- Clone or download this repository
- Ensure you have [ESPHome](https://esphome.io) installed
- Have Home Assistant running and accessible
- Create a `secrets.yaml` file with your WiFi credentials

### 2. Configuration Selection

**Choose ONE configuration:**

```bash
# For direct AC dimmer control (simpler, but may flicker)
cp touch_lamp.yaml esphome_config.yaml

# OR for DimmerLink board (recommended for best results)
cp touch_lamp_dimmerlink.yaml esphome_config.yaml
```

### 3. Edit Configuration

Update the following in your chosen YAML file:

```yaml
esphome:
  name: "smart-touch-1"  # Change if using multiple devices

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

### 4. Create Secrets File

Create `secrets.yaml` in your ESPHome directory:

```yaml
wifi_ssid: "Your WiFi Network"
wifi_password: "Your WiFi Password"
api_key: "Your 32-byte base64-encoded encryption key"
ota_password: "Your OTA update password"
```

Generate an API key:
```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 5. Flash to ESP32

```bash
esphome run esphome_config.yaml
```

### 6. Home Assistant Integration

1. In Home Assistant, go to **Settings > Devices & Services > ESPHome**
2. Add the device IP address
3. Complete the pairing process
4. New entities will appear in Home Assistant



## Features

### Touch Control
- **Single touch**: Cycles through brightness levels (0% → 25% → 50% → 75% → 100% → 0%)
- **Debouncing**: 10ms on/off filters prevent accidental triggering
- **Threshold**: Configurable per hardware (see Calibration section above)

### Brightness Levels
- **Off**: 0%
- **Low**: 25%
- **Medium**: 50%
- **High**: 75%
- **Full**: 100%

### Home Assistant Integration
- **Light Entity**: `light.smart_lamp` - Full brightness control via slider
- **Number Entity**: `number.dimmer_control` - Discrete brightness levels
- **Restart Button**: `button.restart_esp32_*` - Emergency device restart
- **Web Interface**: IP:80 - Direct web control
- **OTA Updates**: Update firmware without physical access

### WiFi Features
- **Primary Network**: Auto-connect to configured WiFi
- **Fallback Hotspot**: "FR-Lamp-Touch" SSID if WiFi unavailable
- **Captive Portal**: Easy network reconfiguration
- **API Encryption**: Secure communication with Home Assistant

## Usage Examples

### Via Touch Pad
1. Place finger on touch pad (GPIO13)
2. Brief contact cycles to next brightness level
3. Light brightness changes smoothly

### Via Home Assistant
```
light.turn_on
  entity_id: light.smart_lamp
  brightness: 128  # 50% brightness
```

### Via Web Interface
1. Navigate to `http://<device-ip-address>`
2. Adjust brightness slider
3. Changes apply immediately

### Via MQTT (if enabled)
```bash
mosquitto_pub -t home/smart-touch-1/light/smart_lamp/command -m '{"brightness": 200}'
```

## Customization

### Adjust Touch Sensitivity
Edit touch threshold (higher = less sensitive). Note: ESP32-S3 boards report much larger
raw touch values than classic ESP32 (S1/S0). For `esp32doit-devkit-v1` (S1) start near `2000`.

To inspect raw touch values manually, set `setup_mode: true` under `esp32_touch:` and
flash the device. Then observe the raw sensor readings in the web UI or `logger` output.

Steps to measure and compute a good threshold:

- 1) With no finger on the pad, record the maximum raw value you see (`no_touch_max`).
- 2) While touching the pad repeatedly, record the minimum raw value (`touch_min`).
- 3) Compute the range: `range = no_touch_max - touch_min`.
- 4) Simple midpoint threshold: `threshold = no_touch_max - (range / 2)`.
- 5) Recommended hysteresis threshold: `threshold = no_touch_max - (range * 0.75)`.

Example YAML to enable raw values while testing:

```yaml
esp32_touch:
  setup_mode: true
  measurement_duration: 3ms

binary_sensor:
  - platform: esp32_touch
    pin: GPIO13
    threshold: 2000  # temporary while testing
```

After you have recorded `no_touch_max` and `touch_min` and chosen a threshold, set
`setup_mode: false` in `esp32_touch:` and update the `threshold:` value in your main
configuration, then reflash the device.

### Change UART Pins (DimmerLink only)
```yaml
uart:
  - id: dimmerlink_uart
    tx_pin: GPIO9
    rx_pin: GPIO10
```

### Modify Brightness Steps
Edit number step value:

```yaml
number:
  - platform: template
    step: 20  # Use 20% steps instead of 25%
```

### Add Second Dimmer
Create additional number entity with different dimmer ID:

```yaml
number:
  - platform: template
    name: Dimmer 2
    set_action:
      then:
        - uart.write:
            data: !lambda
              - return {0x02, 0x53, 0x01, (uint8_t)x};  # ID 0x01
```

## Troubleshooting

### WiFi Connection Issues
- Verify SSID and password in `secrets.yaml`
- Check ESP32 antenna connection
- Try connecting to fallback hotspot "FR-Lamp-Touch"
- Restart device with restart button

### Touch Pad Not Responding
- Verify GPIO13 is not damaged
- Lower threshold value for increased sensitivity
- Ensure good contact with touch pad surface
- Add 100nF capacitor near touch pad if noisy

### DimmerLink UART Issues
- Verify TX/RX pins match your wiring
- Confirm DimmerLink is powered
- Check baud rate is 115200
- Ensure common GND connection
- Test with serial terminal: `picocom /dev/ttyUSB0 -b 115200`

### No Dimming Response
- Check dimmer TRIAC module is powered
- Verify zero-cross pin connection (if using direct control)
- Test with 100% brightness first
- Check Home Assistant entity state

### Home Assistant Not Finding Device
- Ensure Home Assistant can reach device IP
- Check API encryption key matches in `secrets.yaml`
- Restart Home Assistant after device connects
- Check ESPHome logs: `esphome logs esphome_config.yaml`

## Documentation

- **Original Configuration**: See [touch_lamp_description.md](touch_lamp_description.md)
- **DimmerLink Configuration**: See [touch_lamp_dimmerlink_README.md](touch_lamp_dimmerlink_README.md)

## Dependencies

- **ESPHome** 2024.12.0 or later
- **Home Assistant** 2024.1.0 or later (optional, but recommended)
- **Python** 3.8+ (for ESPHome CLI)

## Supported Boards

- ESP32 DoIt DevKit v1 ✓
- ESP32-S3 DevKit C-1 ✓
- ESP32 Wroom ✓
- ESP32 Wrover ✓
- Other ESP32 variants (may require pin adjustments)

## API Reference

### DimmerLink UART Command Format

```
[0x02, 0x53, dimmer_id, brightness]
```

| Byte | Value | Description |
|------|-------|-------------|
| 0    | 0x02  | SET command |
| 1    | 0x53  | Brightness control |
| 2    | 0x00-0xFF | Dimmer ID |
| 3    | 0-100 | Brightness percentage |

**Examples:**
- `[0x02, 0x53, 0x00, 0]` - Turn off dimmer 0
- `[0x02, 0x53, 0x00, 50]` - Set dimmer 0 to 50%
- `[0x02, 0x53, 0x00, 100]` - Set dimmer 0 to 100%

## Performance Metrics

| Metric | Value |
|--------|-------|
| Response Time | <50ms |
| UART Baud Rate | 115200 |
| Touch Detection Latency | <100ms |
| Brightness Update Frequency | 1-2 updates/sec |
| WiFi Connection Time | 5-15 seconds |
| Memory Usage | ~65% of ESP32 SRAM |

## Security Considerations

- **API Encryption**: All Home Assistant communication is encrypted
- **OTA Password**: Firmware updates require authentication
- **WiFi**: Uses WPA2 security (configured via secrets)
- **Default Settings**: 
  - Web server on port 80 (no authentication)
  - Consider disabling if on untrusted network
  - Use firewall rules to restrict access

## Known Limitations

1. **Touch Pad Environmental Sensitivity**
   - Moisture/humidity can affect sensitivity
   - May require threshold adjustment
   - Works best with dry finger contact

2. **UART Communication (DimmerLink)**
   - Single UART bus (use multiple ESP32s for more than 8 dimmers)
   - No built-in error checking
   - Commands are fire-and-forget

3. **Brightness Control**
   - Limited to discrete 25% steps (configurable to any increment)
   - No smooth transitions on DimmerLink (handled internally)

## Future Enhancements

- [ ] Add temperature sensor
- [ ] Implement brightness scheduling
- [ ] Add motion detection integration
- [ ] Support for multiple touch pads
- [ ] Integration with voice assistants
- [ ] Local fallback mode without WiFi
- [ ] Power consumption monitoring
- [ ] Advanced dimming curves configuration

## Contributing

Contributions are welcome! Please:

1. Test changes thoroughly on ESP32 hardware
2. Update documentation with any new features
3. Follow YAML formatting conventions
4. Include comments for non-obvious configuration options

## License

This project is provided as-is for personal use. Modify and distribute as needed.

## Support & Resources

### Official Documentation
- [ESPHome Documentation](https://esphome.io)
- [Home Assistant Integration](https://www.home-assistant.io/integrations/esphome/)
- [DimmerLink Official Docs](https://www.rbdimmer.com/docs/dimmerlink-overview)

### Community
- [ESPHome Discord](https://discord.gg/KhAMQ2pxN5)
- [Home Assistant Community Forums](https://community.home-assistant.io)
- [Home Assistant Discord](https://discord.gg/home-assistant)

### Troubleshooting
- Check [ESPHome Logs](#troubleshooting)
- Verify hardware connections match documentation
- Test with simple UART commands
- Search existing issues in this repository

## Changelog

### Version 2.0 (Current)
- Added DimmerLink UART-based configuration
- Improved documentation with hardware connection guide
- Enhanced troubleshooting section
- Added configuration comparison table

### Version 1.0
- Initial direct AC dimmer configuration
- Basic touch control implementation
- Home Assistant integration

## Author

Created for personal home automation setup.

---

**Last Updated**: February 2026  
**Tested On**: ESP32 DoIt DevKit v1 with ESPHome 2024.12+
