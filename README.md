# Gesture Controlled Robot — ESP32 ESP-NOW

A wireless gesture-controlled robot built with two ESP32 boards communicating over **ESP-NOW** using MAC address pairing. Tilt your hand to drive — no phone, no Wi-Fi network, no app required.

---

## How it works

The system is split into two independent ESP32 units:

**Transmitter** (worn on hand) — reads an MPU-6050 accelerometer/gyroscope module. Tilt forward to move forward, tilt back to reverse, tilt left/right to turn. The gesture data is packed into a struct and broadcast wirelessly via ESP-NOW to the receiver's MAC address.

**Receiver** (mounted on robot) — listens for incoming ESP-NOW packets. On receiving a command, it drives the L298N motor driver accordingly to move the BO motors. No Wi-Fi router or pairing process needed — just power both boards and they talk instantly.

---

## Features

- Wireless range up to ~100m (open space) with zero latency
- No Wi-Fi network or internet required — works completely offline
- Dual-mode: gesture control + optional UART fallback commands
- Tilt threshold tuning via `#define` constants — no recompile needed for most adjustments
- Watchdog on receiver — motors stop automatically if signal is lost for >500ms

---

## Hardware

| Component | Quantity | Notes |
|---|---|---|
| ESP32 Dev Board | 2 | One transmitter, one receiver |
| MPU-6050 (accel + gyro) | 1 | On transmitter hand unit |
| L298N Motor Driver | 1 | On receiver / robot chassis |
| BO Motor + wheel set | 1 set | 2WD differential drive |
| Caster wheel | 1 | Front support |
| 18650 Li-Ion battery (2S) | 2 cells | Powers receiver robot |
| 9V battery or power bank | 1 | Powers transmitter hand unit |
| Acrylic / cardboard chassis | 1 | 2-layer, pre-drilled |
| Jumper wires + breadboard | — | For prototyping |

---

## Wiring

### Transmitter ESP32 ← MPU-6050

| MPU-6050 | ESP32 |
|---|---|
| VCC | 3.3V |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |
| AD0 | GND (I2C address = 0x68) |

### Receiver ESP32 → L298N → Motors

| ESP32 | L298N |
|---|---|
| GPIO 27 | IN1 |
| GPIO 26 | IN2 |
| GPIO 25 | IN3 |
| GPIO 33 | IN4 |
| GPIO 14 | ENA (PWM speed — Motor A) |
| GPIO 12 | ENB (PWM speed — Motor B) |
| GND | GND (shared) |

Battery pack (7.4V) → L298N power input. L298N 5V out → ESP32 VIN.

---

## Software Setup

### Prerequisites

- Arduino IDE 2.x
- ESP32 board package: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
- Libraries (install via Library Manager):
  - `ESP32 ESP-NOW` — built into the ESP32 Arduino core, no separate install
  - `Adafruit MPU6050`
  - `Adafruit Unified Sensor`

### Getting the receiver MAC address

Before flashing the transmitter, you need the receiver's MAC address. Flash this one-liner to the **receiver** ESP32 first:
```cpp
#include <WiFi.h>
void setup() {
  Serial.begin(115200);
  Serial.println(WiFi.macAddress());
}
void loop() {}
```

Open Serial Monitor at 115200 baud. You'll see something like:
```
A4:CF:12:7B:3E:1F
```

Copy this. You'll paste it into the transmitter code in the next step.

---

## Flashing

### Step 1 — Flash the receiver

Open `receiver/receiver.ino` and upload to the receiver ESP32. No changes needed.

### Step 2 — Set MAC address in transmitter

Open `transmitter/transmitter.ino`. Find this line and replace with your receiver's MAC:
```cpp
uint8_t receiverMAC[] = {0xA4, 0xCF, 0x12, 0x7B, 0x3E, 0x1F};
```

Each byte of the MAC address goes as a separate hex value.

### Step 3 — Flash the transmitter

Upload `transmitter/transmitter.ino` to the transmitter ESP32.

### Step 4 — Power both boards

Power on receiver first, then transmitter. The transmitter's onboard LED will blink once on successful ESP-NOW registration. After that — tilt to drive.

---

## Gesture mapping

| Gesture | Action |
|---|---|
| Tilt forward | Move forward |
| Tilt backward | Reverse |
| Tilt left | Turn left |
| Tilt right | Turn right |
| Flat / level | Stop |
| Sharp downward tilt | Emergency stop |

Tilt sensitivity is controlled by `TILT_THRESHOLD` in `transmitter.ino` (default: `15` degrees). Increase to require more tilt, decrease for hair-trigger response.

---

## Project structure
```
gesture-robot/
├── transmitter/
│   └── transmitter.ino      # Runs on hand unit ESP32
├── receiver/
│   └── receiver.ino         # Runs on robot ESP32
├── docs/
│   └── wiring_diagram.png   # Full circuit diagram
└── README.md
```

---

## Troubleshooting

**Robot not responding to gestures**
- Open Serial Monitor on receiver (115200 baud) — it prints every received packet. If nothing shows, the MAC address is wrong.
- Confirm both boards are on the same Wi-Fi channel. ESP-NOW uses Wi-Fi channel 1 by default. If your router is on a different channel, add `WiFi.begin(); WiFi.channel(1);` before `esp_now_init()`.

**Motors running but wrong direction**
- Swap the two wires on the affected motor, or flip the IN1/IN2 logic in `receiver.ino`.

**Jerky / unstable movement**
- Increase `TILT_THRESHOLD` in the transmitter — you may be picking up hand tremor.
- Check that the MPU-6050 VCC is on **3.3V**, not 5V. 5V will not damage it immediately but causes noisy readings on some modules.

**Upload fails after connecting ESP-NOW code**
- ESP-NOW initialises Wi-Fi in station mode. Before uploading new code, hold the BOOT button on the ESP32 while clicking Upload in Arduino IDE.

---

## ESP-NOW — why not Bluetooth or Wi-Fi?

| | ESP-NOW | Bluetooth | Wi-Fi |
|---|---|---|---|
| Latency | ~1ms | ~20–100ms | ~50–200ms |
| Range | ~100m | ~10m | ~50m |
| Needs router | No | No | Yes |
| Pairing process | MAC address only | Pairing handshake | SSID + password |
| Power consumption | Very low | Low | High |

For a real-time robot control application, ESP-NOW is the clear choice. The 1ms latency means the robot feels like a direct physical extension of your hand.

---

## Team

Built as a 6th semester mini-project.

| Role | Responsibility |
|---|---|
| Hardware | Chassis assembly, motor wiring, sensor mounting |
| Firmware — Transmitter | MPU-6050 reading, gesture mapping, ESP-NOW TX |
| Firmware — Receiver | ESP-NOW RX, motor driver control, watchdog |
| Testing & integration | End-to-end testing, tuning, documentation |

---

## License

free to use, modify, and distribute with attribution.
