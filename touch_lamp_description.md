# Smart Touch Lamp Configuration

## Overview
This ESPHome configuration file sets up a **Smart Touch Lamp 1** device based on an ESP32 microcontroller. The device integrates with Home Assistant and features touch-based dimmer control with AC power management.

## Device Information
- **Device Name**: `smart-touch-1`
- **Friendly Name**: Smart Touch Lamp 1
- **Board**: ESP32 DoIt DevKit v1 (with ESP32-S3 DevKit C-1 as commented alternative)
- **Framework**: Arduino

## Connectivity & Security
### WiFi Configuration
- Uses secrets file for SSID and password
- Fallback hotspot enabled: **FR-Lamp-Touch**
- Includes captive portal for easy configuration if WiFi connection fails

### Home Assistant Integration
- API enabled with encryption (key stored in secrets)
- OTA (Over-The-Air) updates enabled via ESPHome platform
- Web server running on port 80

### Logging
- Default logging level: **INFO** (debug level available but commented out)

## Hardware Components & Functionality

### Touch Sensor
- **Platform**: ESP32 Touch
- **Pin**: GPIO13
- **Detection Settings**:
  - Threshold: 550,000
  - Measurement duration: 0.25ms
  - Setup mode: OFF
  - Debouncing: 10ms on/off delay

### Touch Control Behavior
- **Input Type**: Click detection
- **Click Duration Range**: 10ms - 500ms
- **Action**: Cycles through brightness levels (0%, 25%, 50%, 75%, 100%)

### Light/Dimmer Output
- **Light Platform**: Monochromatic
- **Light ID**: `dimmer`
- **Light Name**: Smart Lamp
- **Control Method**: AC Dimmer output
- **Default State**: Always ON (restore mode)

### AC Dimmer Hardware
- **Gate Pin**: GPIO12 (controls the dimmer MOSFET)
- **Zero-Cross Pin**: GPIO5 (synchronization with AC cycle)
- **Brightness Control**: Template number platform with 25% step increments

## Controls & Management
### Dimmer Control (Template Number)
- **Range**: 0 - 100
- **Step Size**: 25
- **Mode**: Optimistic (assumes command success)
- **Brightness Mapping**: Linear (0-100 mapped to 0.0-1.0 brightness)

### System Button
- **Restart Button**: `restart-esp32-dim-touch` - allows remote restart of the ESP32

## Notes & Questions
- A commented section in the light configuration questions whether additional turn-on initialization is needed
- Alternative board configuration (ESP32-S3) is available but currently unused
