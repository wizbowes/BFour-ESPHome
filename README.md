# BFour BF-60 ESPHome Integration

ESPHome firmware for the BFour BF-60 Bluetooth grill thermometer, bridging it to Home Assistant over WiFi.

## What it does

This is an ESPHome config to connect to a BFour BF-60 cooking thermometer. It will probably work with their other devices with a little bit of tweaking.

NOTE: This connects to the base unit NOT the individual temperature probes.

Also included is a separate but quick method to find your device's MAC address if you are struggling.

- Reads temperatures from both probes in real time
- Allows the setting of the Probe Alarm Temp for each probe
- Triggers alarm binary sensors in Home Assistant when a probe reaches its target temperature
- Controls the device alarm sound on/off
- Controls the display unit (Celsius/Fahrenheit)
- Reconnects automatically if the BLE connection drops

## Hardware

- Any ESP32 development board (tested on ESP32-D0WD-V3)
- BFour BF-60 Bluetooth thermometer

## Files

| File | Purpose |
|---|---|
| `bfour-bf60.yaml` | Main firmware — flash this for day-to-day use |
| `bfour-discovery.yaml` | One-time tool for finding your device's MAC address |
| `secrets.yaml.example` | Template for your credentials — copy to `secrets.yaml` |

## Setup

### 1. Create secrets.yaml

Copy `secrets.yaml.example` to `secrets.yaml` and fill in your WiFi credentials. Leave `bfour_mac` for now.

Generate an API encryption key for Home Assistant:

```
python3 -c "import base64,os; print(base64.b64encode(os.urandom(32)).decode())"
```

Paste the output into `secrets.yaml` as `api_encryption_key`. Home Assistant will ask for this key when you add the device.

### 2. Find your device MAC address

The BF-60 uses a BLE MAC address that you need to provide once. The discovery firmware makes this straightforward.

Flash the discovery firmware:

```
esphome upload --device /dev/cu.YOURPORT bfour-discovery.yaml
```

Once it boots and connects to WiFi, open the web interface at `bfour-discovery.local` (or its IP address).

You will see two fields:

- **Device Name Filter** — type part of your device's name. The search is case-insensitive. My BF-60 typically appeared as "Grill BT5.0" as something containing "BF". Note: the probes are also advertised on Bluetooth as "BF-40" (in my case).  Do not use these.  Use the base station.
  
- **Found MAC** — once a matching device is seen, its MAC address appears here.  

Hold the base unit close to the ESP32 to get a strong signal. The field updates to the closest matching device, so if multiple devices match your filter, the one physically nearest will win.

Copy the MAC address into `secrets.yaml`:

```yaml
bfour_mac: "XX:XX:XX:XX:XX:XX"
```

### 3. Flash the main firmware

```
esphome upload --device /dev/cu.YOURPORT bfour-bf60.yaml
```

### 4. Add to Home Assistant

The device will be discovered automatically via the ESPHome integration. After adding it, set your probe target temperatures in the UI — these are stored on the ESP32 and survive reboots, but are wiped if you reflash.

## Entities

| Entity | Type | Description |
|---|---|---|
| Probe - Black | Sensor | Black probe temperature |
| Probe - White | Sensor | White probe temperature |
| Probe Black Alarm | Binary Sensor | Goes active when black probe reaches target |
| Probe White Alarm | Binary Sensor | Goes active when white probe reaches target |
| Probe Black Target | Number | Target temperature for black probe alarm (1-99 in C, 1-210 in F) |
| Probe White Target | Number | Target temperature for white probe alarm (1-99 in C, 1-210 in F) |
| Display Celsius | Switch | Toggle device display between C and F — existing targets are converted automatically |
| Alarm Sound | Switch | Enable or disable the device alarm beeper |
| BFour Signal | Sensor | BLE signal strength in dBm |

## Notes

- Target temperatures are stored on the ESP32 and survive reboots, but are wiped if you reflash. Re-set them after flashing.
- The alarm is evaluated entirely by the ESP32, not the base unit. Each temperature update either sets or clears the alarm based on whether the probe is above or below the target.
- The alarm clears as soon as the probe drops below the target.
- Switching between C and F converts the existing target values automatically and resends them to the device.
- The 4-minute keep-alive write prevents the device from going to sleep while connected.
- If the BLE connection drops, the ESP32 reconnects automatically and the alarm sensors reset to off.
