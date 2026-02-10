# Touch Pad Calibration Helper Configuration

This is a temporary configuration designed to help you determine the optimal threshold value for your ESP32 touch pad. It does not control the dimmer and runs in setup mode to provide detailed feedback about your touch pad readings.

## Purpose

Finding the right touch pad threshold is critical for reliable operation:
- **Too high**: The pad won't register touches (unresponsive)
- **Too low**: The pad triggers false positives when not being touched
- **Just right**: Responsive but stable

This helper guides you through a calibration process to find the optimal value.

## Hardware Requirements

Same as the main project:
- ESP32 microcontroller
- Capacitive touch pad on GPIO13
- Final hardware configuration (dimmer, cabling, etc.)

## How to Use

### Step 1: Flash the Calibration Configuration

Replace your current configuration with `touch_lamp_calibration_helper.yaml` and flash it to your ESP32:

```bash
esphome run touch_lamp_calibration_helper.yaml
```

The device will boot into **setup mode**, which means:
- Touch sensor readings are continuous and detailed
- You'll see raw ADC values updating in real-time
- No dimmer control happens

### Step 2: Start Calibration

Once the device is running, access it via:
- **Home Assistant**: Look for the "Start Calibration" button in the device entities
- **Web Interface**: Navigate to http://<esp32-ip>/

Click the **"Start Calibration"** button to begin Phase 1.

### Phase 1: Testing NO TOUCH (10-15 seconds)

After clicking "Start Calibration":

1. **Keep your hand completely away** from the touch pad
2. Let the sensor settle and establish baseline readings
3. Watch the debug log output - you'll see: `Testing NO TOUCH - Keep pad clear`
4. The "No Touch Max Value" sensor will update with the maximum value detected when the pad is not being touched
5. Wait until the readings stabilize (typically 10-15 seconds)
6. Click **"End No-Touch Phase (Start Touch Phase)"** when ready

**What's happening**: The system records the highest ADC value when the pad is NOT being touched. This establishes your "idle" baseline.

### Phase 2: Testing TOUCH (10-15 seconds)

After clicking "End No-Touch Phase":

1. **Repeatedly touch the touch pad** - tap it many times
2. Try different touch styles:
   - Quick taps
   - Longer presses (1-2 seconds)
   - Various finger positions
   - Different finger areas (tip, pad, side)
3. Watch the debug log - you'll see: `Testing TOUCH - Repeatedly touch pad`
4. The "Touch Min Value" sensor will update with the minimum value detected when actively touching the pad
5. Continue for 10-15 seconds to ensure you capture your true touch range
6. Click **"Complete Calibration"** when done

**What's happening**: The system records the lowest ADC value when you actively touch the pad. This establishes your "touch" state range.

### Step 3: Review Results

After clicking "Complete Calibration":

1. Check the debug log output - it will show:
   - No-Touch Maximum Value: `XXXX` (idle baseline)
   - Touch Minimum Value: `YYYY` (active touch baseline)
   - **Calculated Threshold: `ZZZZ`** ← Use this value!

2. These values are also available as sensor entities:
   - "No Touch Max Value"
   - "Touch Min Value"
   - "Recommended Threshold"

### Step 4: Apply the Threshold

Once you have your recommended threshold value:

1. Open your main configuration file (`touch_lamp.yaml` or `touch_lamp_dimmerlink.yaml`)
2. Find the touch sensor configuration:
   ```yaml
   binary_sensor:
     - platform: esp32_touch
       pin: GPIO13
       threshold: 550000  # ← Replace this value
   ```
3. Replace `550000` with your calculated threshold
4. Also set `setup_mode: false` in the `esp32_touch:` section
5. Reflash the device with your main configuration

## Understanding the Numbers

### Raw Touch Pad Readings

The ESP32 touch sensor outputs a capacitance value (ADC count):

- **Higher numbers** = less capacitance = **NOT touching** (open pad)
- **Lower numbers** = more capacitance = **TOUCHING** (pad loaded with finger)

### Threshold Calculation

The helper uses this formula for the recommended threshold:

```
Threshold = No-Touch-Max - (Range × 75%)
```

Where:
- `No-Touch-Max` = the maximum value when pad is not touched
- `Range` = difference between no-touch and touch readings
- `75%` = hysteresis factor (75% of the way down from idle to touch)

This provides good hysteresis (prevents bouncing between states) while being responsive to touches.

### Example

If calibration shows:
- No Touch Max: `630000`
- Touch Min: `450000`
- Range: `630000 - 450000 = 180000`
- Threshold: `630000 - (180000 × 0.75) = 630000 - 135000 = 495000`

You would use `threshold: 495000` in your configuration.

## Troubleshooting

### Pad doesn't register in Phase 1 or 2

- Check that the touch pad is properly wired to GPIO13
- Verify the pad is exposed and accessible (not covered by thick material)
- Make sure there's no moisture or conductive residue on the pad

### Values don't change between phases

- Ensure the pad has good electrical continuity
- Try touching firmer or with more surface area
- Check for loose wires or bad solder joints

### Very high variance between readings

- This is normal during calibration - the system captures the extremes
- If readings jump 100,000+ units between touches, your pad may be:
  - Too sensitive (picking up air motion)
  - Poorly shielded (EM interference)
  - In a high-humidity environment

### Threshold seems too high or too low after applying it

- Run calibration again - press "Reset Calibration" first
- Repeat with more deliberate touch patterns in Phase 2
- Try different finger pressure and contact area

## Advanced: Manual Threshold Adjustment

If the automatic calculation isn't perfect, you can adjust manually:

- **If threshold rejects real touches**: Increase the threshold value (e.g., `495000` → `510000`)
- **If threshold detects false positives**: Decrease the threshold value (e.g., `495000` → `480000`)

Adjust in increments of 10,000-20,000 until you find the sweet spot.

## Reverting to Production

Once you're satisfied with your threshold:

1. Flash your main configuration (`touch_lamp.yaml` or `touch_lamp_dimmerlink.yaml`)
2. The dimmer control and full features will be restored
3. The calibration helper can be kept as a backup for future threshold adjustments

## Additional Notes

- **Setup Mode**: Calibration config uses `setup_mode: true` which shows raw values continuously. Your production config should use `setup_mode: false`
- **Temporary State**: This config is meant as a temporary tool - not for permanent use
- **No Dimmer Control**: This config intentionally excludes light/output components
- **Final Hardware**: Use in your final hardware configuration to get accurate readings (including cabling, housing, etc.)

## FAQ

**Q: Why do the readings change so much?**
A: The touch sensor is very sensitive. Humidity, temperature, and even nearby objects affect readings. This is why calibration is important!

**Q: Can I use the same threshold forever?**
A: Usually yes, but if you change the pad, housing, or wiring, recalibrate. Seasonal humidity changes might require adjustment.

**Q: What if "No Touch Max" is lower than "Touch Min"?**
A: This means your pad readings are inverted or there's a hardware issue. Check GPIO13 connection and pad wiring.

**Q: How do I know when Phase 1/2 is done?**
A: When the value in the log stabilizes and stops increasing/decreasing significantly (usually 10-15 seconds). You can also watch the sensor value in Home Assistant or the web interface.
