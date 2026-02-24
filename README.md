# Smart Touch Lamp with ESPHome & DimmerLink UART

## Overview

This project implements a flicker-free, touch-controlled AC lamp dimmer using ESPHome on an ESP32 microcontroller and a DimmerLink UART board. The system integrates seamlessly with Home Assistant, providing robust remote and physical control, custom brightness cycling, and reliable state synchronization.

## Features

- Capacitive touch pad cycles through customizable brightness levels (e.g., 0%, 30%, 50%, 100%)
- Flicker-free dimming via DimmerLink UART board
- Home Assistant integration: lamp appears as a standard light entity
- State synchronization between physical touch and Home Assistant
- Restart button for ESP32 maintenance
- **Note:** After firmware upgrades, power-cycle both ESP32 and DimmerLink for reliable operation

## Hardware Requirements

- **ESP32**: DoIt DevKit v1 or ESP32-S3 DevKit C-1 or higher
- **DimmerLink UART Board**: Dedicated MCU for AC dimmer control *and*
- **AC Dimmer Module**: TRIAC-based, controlled by DimmerLink *or*
- **Combination board**: Containing the DimmerLink and AC Dimmer
- **Touch Sensor**: Capacitive touch pad (GPIO13) usually the frame of the lamp
- **Power Supply**: ESP32 (5V/1A USB or 3.3V), DimmerLink (5V or 3.3V), AC dimmer (110/230V)

### Wiring Summary

| ESP32 Pin | DimmerLink Pin | Purpose |
|-----------|---------------|---------|
| GPIO17    | RX            | UART TX to DimmerLink |
| GPIO16    | TX            | UART RX from DimmerLink |
| GPIO13    | Touch Pad     | User input |
| GND       | GND           | Common ground |
| 3.3V/5V   | VCC           | Power |

## Project Structure

```
Touch Lamp/
├── README.md                         # This file
├── touch_lamp.yaml                   # Original config (direct AC dimmer)
├── touch_lamp_description.md         # Original config docs
├── touch_lamp_dimmerlink.yaml        # ESPHome config for DimmerLink UART
└── touch_lamp_dimmerlink_README.md   # DimmerLink config docs
```

## ESPHome Configuration Highlights

- **Touch Sensor**: Cycles through a custom list of brightness values, always starting from the closest current value (and treating off as 0%).
- **Number Entity**: Triggers light brightness changes; all state logic flows through the light entity for Home Assistant compatibility.
- **Light Entity**: Monochromatic light, synchronized with number entity and touch input.
- **Output Block**: Sends UART commands to DimmerLink based on light brightness.
- **Logging**: Debug logs for brightness changes and UART communication.
- **Restart Button**: For ESP32 maintenance.

## Home Assistant Integration

- Lamp appears as a standard light entity
- Touch pad acts as a physical interface for cycling brightness with custom brightness steps
- All state changes (touch, Home Assistant, automations) are synchronized

## Credits & Links

- DimmerLink board: [RBDimmer.com](https://www.rbdimmer.com/shop/dimmerlink-controller-uart-i2c-interface-for-ac-dimmers-48)
- ESPHome: [esphome.io](https://esphome.io/)
- Home Assistant: [home-assistant.io](https://www.home-assistant.io/)

## Quick Start

### 1. Preparation

- Clone or download this repository
- Ensure you have [ESPHome](https://esphome.io) installed
- Have Home Assistant running and accessible
- Create a `secrets.yaml` file with your WiFi credentials

## Usage Examples

### Via Touch Pad
1. Place finger on touch pad (GPIO13)
2. Brief contact cycles to next brightness level

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

## Customization

### Customizable Brightness Levels

Adjust brightness levels in the touch sensor lambda (e.g., `{0, 30, 50, 100}`)

### Adjust Touch Sensitivity
Edit touch threshold (higher = less sensitive). Note: ESP32-S3 boards report much larger
raw touch values than classic ESP32 (S1/S0). For `esp32doit-devkit-v1` (S1) start near `2000`.

To inspect raw touch values manually, set `setup_mode: true` under `esp32_touch:`, 
`level: debug` under `logger:` and flash the device. Then observe the raw sensor readings
in the web UI or `logger` output.

Steps to measure and compute a good threshold:

- 1) With no finger on the pad, record the maximum raw value you see (`no_touch_max`).
- 2) While touching the pad repeatedly, record the minimum raw value (`touch_min`).
- 3) Compute the range: `range = no_touch_max - touch_min`.
- 4) Simple midpoint threshold: `threshold = no_touch_max - (range / 2)`.
- 5) Recommended hysteresis threshold: `threshold = no_touch_max - (range * 0.75)`.

Example YAML to enable raw values while testing:

```yaml
logger:
  level: debug

esp32_touch:
  setup_mode: true

binary_sensor:
  - platform: esp32_touch
    pin: GPIO13
    threshold: 2000  # temporary while testing
```

After you have recorded `no_touch_max` and `touch_min` and chosen a threshold, set
`setup_mode: false` in `esp32_touch:` and update the `threshold:` value in your main
configuration, then reflash the device.

## Troubleshooting

- **Power-cycle after upgrades**: Both ESP32 and DimmerLink may require a restart
- **Touch not responsive**: Adjust threshold or check wiring
- **No dimming response**: Verify UART wiring, baud rate, and DimmerLink power
- **Debug logs**: Use logger output for troubleshooting

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

## Future Enhancements

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
