# Smart Touch Lamp with DimmerLink UART & ESPHome

## Overview

This configuration uses ESPHome on an ESP32 to control an AC lamp dimmer via the DimmerLink UART board. The system provides flicker-free dimming, custom brightness cycling, and robust Home Assistant integration. All state changes are synchronized between touch input and Home Assistant, and custom brightness levels are easily adjustable.

## Key Features

- Touch pad cycles through custom brightness levels (e.g., 0%, 30%, 50%, 100%)
- Flicker-free dimming via DimmerLink UART board
- Home Assistant integration: lamp appears as a standard light entity
- Customizable brightness steps and cycling logic
- State synchronization between touch, Home Assistant, and automations
- OTA updates and fallback WiFi hotspot
- Debug logging for troubleshooting
- Restart button for ESP32 maintenance
- **Note:** After firmware upgrades, power-cycle both ESP32 and DimmerLink for reliable operation

## Hardware Connections

### ESP32 to DimmerLink Board (UART)

| ESP32 Pin | DimmerLink Pin | Purpose |
|-----------|---------------|---------|
| GPIO17    | RX            | UART TX to DimmerLink |
| GPIO16    | TX            | UART RX from DimmerLink |
| GPIO13    | Touch Pad     | User input |
| GND       | GND           | Common ground |
| 3.3V/5V   | VCC           | Power |

### DimmerLink to AC Dimmer Module

- Gate control: triggers TRIAC for phase angle control
- Zero-cross detection: synchronizes with AC mains
- AC Load: lamp or other AC device

## ESPHome Configuration Highlights

- Touch sensor lambda cycles through a custom list of brightness values, always starting from the closest current value (and treating off as 0%).
- Number entity triggers light brightness changes; all state logic flows through the light entity for Home Assistant compatibility.
- Light entity: monochromatic, synchronized with number entity and touch input.
- Output block: sends UART commands to DimmerLink based on light brightness.
- Logging: debug logs for brightness changes and UART communication.
- Restart button: for ESP32 maintenance.

## Example YAML Snippet

```yaml
binary_sensor:
  - platform: esp32_touch
    name: "Touch Pad Pin 13"
    pin: GPIO13
    threshold: 700
    on_click:
      - min_length: 10ms
        max_length: 500ms
        then:
          - lambda: |-
              static const std::vector<int> levels = {0, 30, 50, 100};
              int current = 0;
              if (id(dimmer).current_values.is_on()) {
                current = (int)(id(dimmer).current_values.get_brightness() * 100.0f);
              } else {
                current = 0;
              }
              int closest_idx = 0;
              int min_diff = abs(current - levels[0]);
              for (size_t i = 1; i < levels.size(); ++i) {
                int diff = abs(current - levels[i]);
                if (diff < min_diff) {
                  min_diff = diff;
                  closest_idx = i;
                }
              }
              int next_idx = (closest_idx + 1) % levels.size();
              int next = levels[next_idx];
              auto call = id(dimmer_control).make_call();
              call.set_value(next);
              call.perform();

number:
  - platform: template
    id: dimmer_control
    min_value: 0
    max_value: 100
    step: 1
    internal: true
    set_action:
      then:
        - light.turn_on:
            id: dimmer
            brightness: !lambda 'return (float)x / 100.0f;'

light:
  - platform: monochromatic
    id: dimmer
    name: "Smart Lamp"
    output: dimmerlink_output

output:
  - platform: template
    id: dimmerlink_output
    type: float
    write_action:
      then:
        - uart.write:
            id: dimmerlink_uart
            data: !lambda |-
              uint8_t brightness = (uint8_t)(state * 100.0);
              return {0x02, 0x53, 0x00, brightness};
```

## Home Assistant Integration

- Lamp appears as a standard light entity with custom brightness steps
- Touch pad acts as a physical interface for cycling brightness
- All state changes (touch, Home Assistant, automations) are synchronized
- Restart button for device maintenance

## Customization

- Adjust brightness levels in the touch sensor lambda (e.g., `{0, 30, 50, 100}`)
- Change touch threshold for sensitivity
- Modify UART pins if needed
- Add more dimmers by duplicating number/light/output blocks with new IDs

## Troubleshooting

- **Power-cycle after upgrades**: Both ESP32 and DimmerLink may require a restart
- **Touch not responsive**: Adjust threshold or check wiring
- **No dimming response**: Verify UART wiring, baud rate, and DimmerLink power
- **Debug logs**: Use logger output for troubleshooting

## Credits & Links

- DimmerLink board: [RBDimmer.com](https://www.rbdimmer.com/shop/dimmerlink-controller-uart-i2c-interface-for-ac-dimmers-48)
- ESPHome: [esphome.io](https://esphome.io/)
- Home Assistant: [home-assistant.io](https://www.home-assistant.io/)
- **8N1 format**: 8 data bits, no parity, 1 stop bit
- **Pins**: Adjust GPIO17/GPIO16 if using different UART pins on your ESP32

### 2. Touch Pad Configuration

```yaml
esp32_touch:
  setup_mode: false
  measurement_duration: 0.25ms
```

The touch pad on **GPIO13** detects when the user touches the pad and cycles through brightness levels.

Quick testing tip: to view raw touch values while you test, set `setup_mode: true` temporarily:

```yaml
esp32_touch:
  setup_mode: true
```

Then watch the web UI or `logger` output to capture a `no_touch` (idle) maximum and a `touch` minimum.
Calculate `range = no_touch_max - touch_min` and pick a threshold at the midpoint or closer to the
idle side for hysteresis (e.g. `threshold = no_touch_max - (range * 0.75)`). Remember to set
`setup_mode: false` and restore your chosen `threshold:` before putting the device back into
normal operation.

**Behavior:**
- Short touch (10-500ms): Increments brightness to next level (0% → 25% → 50% → 75% → 100% → 0%)
- Uses debouncing filters to prevent spurious triggers

### 3. Brightness Control Number

The `dimmer_control` number entity ranges from 0-100 in 25% increments:
- **0**: Off
- **25**: 25% brightness
- **50**: 50% brightness
- **75**: 75% brightness
- **100**: Full brightness

### 4. UART Commands to DimmerLink

The configuration sends commands to DimmerLink using the UART protocol:

```
Command Format: [0x02, 0x53, dimmer_id, brightness]
```

| Byte | Value | Meaning |
|------|-------|---------|
| 0    | 0x02  | Command type (SET) |
| 1    | 0x53  | Brightness command |
| 2    | 0x00  | Dimmer ID (0 = first dimmer) |
| 3    | 0-100 | Brightness percentage |

**Example Commands:**
- `[0x02, 0x53, 0x00, 0]` → Set dimmer 0 to 0% (off)
- `[0x02, 0x53, 0x00, 50]` → Set dimmer 0 to 50%
- `[0x02, 0x53, 0x00, 100]` → Set dimmer 0 to 100% (full)

### 5. Light Entity

The `dimmer` light entity provides Home Assistant with a standard brightness-controllable light:
- **Name**: "Smart Lamp"
- **Control method**: UART commands via the `dimmerlink_output`
- **Restore mode**: Always powers on (ALWAYS_ON)

## Operational Flow

1. **User touches the touch pad** on GPIO13
2. **ESP32 detects the touch** and increments the brightness level
3. **UART command is sent** to DimmerLink with the new brightness value
4. **DimmerLink processes** the command and adjusts the TRIAC phase angle
5. **Lamp brightness changes** smoothly without flicker

## Home Assistant Integration

Once the device is added to Home Assistant:

- **Light control** via `light.smart_lamp` entity
- **Brightness adjustment** via slider (0-100%)
- **Restart button** for emergency reboot
- **Touch input** visible as a binary sensor (optional - define if needed)
- **Web interface** accessible at the device's IP address on port 80

## Advantages Over Direct AC Dimmer Control

| Feature | Direct `ac_dimmer` | DimmerLink (UART) |
|---------|-------------------|-------------------|
| Flicker | May flicker due to timing conflicts | Flicker-free (dedicated MCU) |
| Library complexity | Complex interrupt handling | Simple 4-byte commands |
| Update flexibility | Tied to ESPHome library changes | Firmware updates via DimmerLink |
| Multi-device support | Single dimmer per ESP32 | Multiple dimmers on same UART |
| Reliability | Depends on ESP32 load | Independent of ESP32 CPU load |

## Customization

### Adjusting Touch Threshold

If the touch pad is too sensitive or not sensitive enough, modify:

```yaml
binary_sensor:
  - platform: esp32_touch
    # For ESP32 S1/S0 boards (esp32doit-devkit-v1) a good starting value
    # is around 2000. Increase to 3000+ for less sensitivity.
    threshold: 2000  # Adjust this value (higher = less sensitive)
```

### Changing UART Pins

To use different ESP32 pins for UART communication:

```yaml
uart:
  - id: dimmerlink_uart
    tx_pin: GPIO9    # Change to your TX pin
    rx_pin: GPIO10   # Change to your RX pin
```

### Adding More Dimmers

To control multiple dimmers, create additional number entities with different dimmer IDs:

```yaml
number:
  - platform: template
    name: Dimmer Control 2
    id: dimmer_control_2
    min_value: 0
    max_value: 100
    step: 25
    set_action:
      then:
        - uart.write:
            id: dimmerlink_uart
            data: !lambda
              - return {0x02, 0x53, 0x01, (uint8_t)x};  # ID changed to 0x01
```

### Brightness Step Changes

To use different increment steps (e.g., 20% instead of 25%):

```yaml
number:
  - platform: template
    step: 20  # Now cycles through 0%, 20%, 40%, 60%, 80%, 100%
```

## Troubleshooting

### UART Communication Issues

1. **Check pins**: Verify GPIO17/GPIO16 connections to DimmerLink TX/RX
2. **Baud rate mismatch**: Ensure DimmerLink is set to 115200 baud
3. **Ground connection**: Confirm GND is connected between ESP32 and DimmerLink
4. **Logic level**: ESP32 outputs 3.3V; DimmerLink should accept this voltage

### No Dimming Response

1. **Verify UART is working**: Enable debug logging and check for transmission
2. **Check DimmerLink power**: Ensure DimmerLink has adequate power supply
3. **Test with serial terminal**: Send raw UART commands to verify DimmerLink responds
4. **Dimmer ID**: Confirm you're using the correct dimmer ID (default is 0x00)

### Touch Pad Not Working

1. **Threshold too high**: Lower the threshold value to make it more sensitive
2. **No contact**: Ensure the touch pad is connected properly
3. **Electrical noise**: Add a capacitor near the touch pad (typically 100nF)

## Files Reference

- **Configuration file**: `touch_lamp_dimmerlink.yaml`
- **Device name**: `smart-touch-1-dimmerlink`
- **Original configuration**: `touch_lamp.yaml` (uses direct AC dimmer control)

## Security Notes

- API encryption key stored in `secrets.yaml`
- WiFi credentials stored in `secrets.yaml`
- OTA update password stored in `secrets.yaml`
- Web server on port 80 (no authentication by default - consider enabling)

## Next Steps

1. Wire the ESP32 to the DimmerLink board according to the connections table above
2. Verify DimmerLink board is powered and set to UART mode
3. Flash this configuration to your ESP32
4. Test the touch pad and UART communication in Home Assistant
5. Adjust touch threshold and timing parameters as needed for your application
