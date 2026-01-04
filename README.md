# Smart Irrigation System (ESP32 + Firebase + Web App)

A smart irrigation project using **ESP32**, **capacitive soil moisture sensor**, **HC-SR04 water level/volume**, **relay + pump**, **SH1106 OLED**, and **Firebase Realtime Database**.  
The web app allows selecting watering modes and adjusting parameters in real time.

---

## Features

- **Soil moisture monitoring (Soid %)** using Capacitive Soil Moisture Sensor v1.2 (AO → ADC).
- **Water remaining estimation (Volume ml)** using HC-SR04 and volume calculation.
- **OLED SH1106 display**:
  - Soil moisture (%)
  - Water volume (ml)
  - Bottom fixed line: `Th:<threshold>%  M:<mode>`
- **Low water protection**:
  - If `Volume < 200 ml` → **pump is blocked** (forced OFF).
  - Warn LED blinks **500ms ON / 500ms OFF** (same logic as original).
- **3 watering modes controlled from web/Firebase**:
  1. **MANUAL** (Mode = 0)  
     - Pump is direct **ON/OFF** (no threshold, no timer).
     - **Physical button** and **web switch** work in parallel and stay **synced**.
  2. **AUTO** (Mode = 1)  
     - If `Soid < Threshold` → pump runs for `PumpSeconds` then stops.
     - Repeats after a cooldown delay if still dry (configurable in code).
  3. **SCHEDULE** (Mode = 2)  
     - Runs at scheduled date/time for `PumpSeconds` then stops.
- **Serial Monitor output** every **5 seconds** (same values as web):
  - Soid
  - Volume
  - Threshold
  - Mode
  - Separator: `----`

---

## Hardware

### ESP32 Pins

| Module | Signal | ESP32 GPIO |
|------|--------|------------|
| Relay (Pump) | IN | GPIO27 |
| Pump LED | LED | GPIO16 |
| Button | INPUT_PULLUP | GPIO32 |
| HC-SR04 | TRIG | GPIO5 |
| HC-SR04 | ECHO | GPIO18 |
| Low water warn LED | LED | GPIO17 |
| Soil sensor v1.2 | AO | GPIO34 |
| OLED SH1106 I2C | SDA | GPIO21 |
| OLED SH1106 I2C | SCL | GPIO22 |

### Power notes (important)

- **ESP32 is 3.3V logic**.
- **HC-SR04 ECHO is 5V** if HC-SR04 is powered by 5V → you **must use a level shifter / voltage divider** on ECHO to protect ESP32.
- Soil sensor v1.2 can be powered by **5V**, but its AO voltage must still be safe for ESP32 ADC.  
  (Most versions output within 0–3.3V, but you should verify with a multimeter. If it can exceed 3.3V, use a divider.)

---

## Firebase Realtime Database Structure

Base path: **`/Var`**

| Key | Type | Description |
|-----|------|-------------|
| `Soid` | int | Soil moisture (%) from ESP32 |
| `Volume` | float / int | Water remaining (ml) |
| `Mode` | int | 0=MANUAL, 1=AUTO, 2=SCHEDULE |
| `ManualSwitch` | int | 0/1 (MANUAL ON/OFF), synced with physical button |
| `Threshold` | int | Soil threshold (%) used in AUTO |
| `PumpSeconds` | int | Pump run time (seconds) used in AUTO + SCHEDULE |
| `Schedule/Date` | string | `YYYY-MM-DD` |
| `Schedule/Time` | string | `HH:MM` |

---

## Steps

1. **Wire the hardware**
   - Connect sensors, relay, OLED, button exactly as listed in the pin table.
   - Ensure **common GND** between ESP32, sensors, and relay module.
   - If pump/relay uses a separate supply, connect grounds together.

2. **Create Firebase Realtime Database**
   - Create a Firebase project
   - Enable **Realtime Database**
   - Create (or ensure) path: `/Var`
   - Add keys as needed: `Mode`, `ManualSwitch`, `Threshold`, `PumpSeconds`, `Schedule/Date`, `Schedule/Time`
   - Set database rules (temporary for testing) to allow read/write if needed.

3. **Install required libraries**
   - `FirebaseESP32`
   - `Adafruit_GFX`
   - `Adafruit_SH110X`

4. **Configure credentials**
   - In ESP32 firmware, set:
     - `WIFI_SSID`, `WIFI_PASSWORD`
     - `FIREBASE_HOST`, `FIREBASE_AUTH`
     - `path = "/Var"`

5. **Upload firmware to ESP32**
   - Open Serial Monitor at **115200 baud**
   - Confirm the device connects to WiFi and starts updating.

6. **Open the web app**
   - The web app reads/writes `/Var` keys to control modes, thresholds, and timers.
   - Verify that switching modes changes ESP32 behavior.

---

## Volume calculation

This project estimates remaining water volume (ml) using an HC-SR04 ultrasonic sensor and a container model.

### 1) Distance from HC-SR04
- Trigger pulse: 10 µs
- Measure echo duration using `pulseIn(ECHO, HIGH, timeout)`
- Convert to distance (cm):

`distanceCm = duration * SOUND_SPEED / 2`

Where:
- `SOUND_SPEED = 0.034` cm/µs

### 2) Water height
Water height in the container:

`h = CUP_HEIGHT - distanceCm`

Clamp to valid range:
- if h < 0 → h = 0
- if h > MAX_H → h = MAX_H

### 3) Radius model
The container is modeled as a frustum-like shape where radius changes with height.

`radius = rBottom + (rTop - rBottom) * (h / MAX_H)`

Used values in code:
- rBottom = 3.0 (cm)
- rTop    = 5.5 (cm)

### 4) Volume formula + scaling
Using the same formula as original code:

`volume = (1/3) * PI * h * (rBottom^2 + 3*rCurrent + rCurrent^2)`

Then scale to a maximum volume:

`volume = volume * (800 / volumeMax)`

Where `volumeMax` is computed at MAX_H using rTop.

### 5) Low water logic
Low water is detected exactly as original:

`lowWater = (volume < 200)`

If `lowWater` is true:
- Pump is blocked (forced OFF)
- Warn LED blinks 500ms ON / 500ms OFF

---

## Getting Started

1. Create Firebase Realtime Database.
2. Set your WiFi and Firebase credentials in code:
   - `WIFI_SSID`, `WIFI_PASSWORD`
   - `FIREBASE_HOST`, `FIREBASE_AUTH`
3. Upload code to ESP32.
4. Open Serial Monitor (115200 baud) to verify values.
5. Control via Web App (reads/writes `/Var`).

---

## Troubleshooting

### ESP32 resets / watchdog
- Check power supply (pump relay noise can reset ESP32).
- Use separate power for pump and ESP32 with common GND.
- Add proper relay module, flyback diode (if needed), and decoupling capacitors.
- Keep wires short and avoid running pump power lines near signal lines.

### No Firebase updates
- Confirm WiFi connected.
- Check Firebase host/auth and database path `/Var`.
- Ensure database rules allow write (or use correct legacy token).
- Avoid calling Firebase set/get from multiple tasks with the same FirebaseData object.

### HC-SR04 unstable readings
- Ensure ECHO level shifting if HC-SR04 is powered at 5V.
- Keep sensor stable, avoid vibration.
- Make sure container geometry constants match your real container.

### Soil sensor values not stable
- Check AO wiring and power.
- Recalibrate `rawDry` and `rawWet`.
- If sensor output can exceed 3.3V, use a voltage divider.

---