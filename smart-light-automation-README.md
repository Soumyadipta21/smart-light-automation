# Smart Light Automation System

An Arduino-based presence-sensing lighting system that automatically turns a light on when a person is nearby and turns it off after they leave — with a built-in manual "Sleep Mode" override.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Hardware Components](#hardware-components)
- [Circuit Diagram / Pin Connections](#circuit-diagram--pin-connections)
- [How It Works](#how-it-works)
- [Software Architecture](#software-architecture)
- [Code Walkthrough](#code-walkthrough)
- [Setup & Installation](#setup--installation)
- [Usage](#usage)
- [Challenges & Solutions](#challenges--solutions)
- [What I Learned](#what-i-learned)
- [Future Improvements](#future-improvements)
- [License](#license)

---

## Overview

Most lights in a home or workspace rely on manual switching — someone has to remember to turn them on when entering a room and off when leaving. This is both inconvenient and a minor source of wasted electricity when people forget.

This project solves that with a simple, low-cost automation circuit built around an **Arduino Uno** and an **HC-SR04 ultrasonic distance sensor**. The system continuously measures the distance to the nearest object in front of it. If a person comes within a configurable range, the light turns on immediately. When the person leaves, instead of switching off right away (which would cause the light to flicker if the person briefly steps out of range), the system waits **30 seconds** before turning off — a grace period that makes the automation feel natural instead of jumpy.

A **Sleep Mode**, toggled by a physical switch, lets the user disable all automatic behavior and force the light off — useful at night or whenever the automation isn't wanted.

This isn't just a relay wired to a motion sensor. The core engineering problem here was **timing without blocking** — the Arduino has to keep sensing distance continuously, even while "waiting" to turn the light off, which rules out the naive approach of using `delay()`.

---

## Features

- **Instant-on detection** — light turns on the moment someone enters range
- **Debounced auto-off** — 30-second grace period prevents flicker from brief or repeated movement at the edge of detection range
- **Non-blocking timing** — sensor keeps reading continuously; the Arduino is never "frozen" waiting for the timer
- **Sleep Mode override** — a physical switch disables automation entirely, forcing the light off regardless of sensor input
- **Configurable detection range** — adjust a single constant to match room size
- **State-machine design** — clean, predictable transitions between ON / OFF / SLEEP states

---

## Hardware Components

| Component | Purpose | Approx. Qty |
|---|---|---|
| Arduino Uno / Nano | Microcontroller running the logic | 1 |
| HC-SR04 Ultrasonic Sensor | Measures distance to detect presence | 1 |
| Relay Module (or LED for testing) | Switches the actual light load | 1 |
| SPDT Toggle Switch | Enables/disables Sleep Mode | 1 |
| Jumper Wires | Connections | ~15 |
| Breadboard | Prototyping | 1 |
| 5V Power Supply / USB cable | Power for Arduino | 1 |

> **Note:** If controlling an actual mains-voltage light (not a low-voltage LED), the relay module must be rated for mains switching, and mains wiring should only be done with proper safety precautions/insulation.

---

## Circuit Diagram / Pin Connections

| Arduino Pin | Connects To |
|---|---|
| Pin 9 | HC-SR04 `Trig` |
| Pin 10 | HC-SR04 `Echo` |
| Pin 8 | Relay/LED (light control, via relay `IN` pin) |
| Pin 7 | Sleep Mode switch (other leg to GND) |
| 5V | HC-SR04 `VCC`, Relay `VCC` |
| GND | HC-SR04 `GND`, Relay `GND`, Switch common |

The Sleep Mode switch uses the Arduino's internal pull-up resistor (`INPUT_PULLUP`), so it reads `HIGH` by default and `LOW` when the switch is closed to GND — no external resistor needed.

---

## How It Works

### 1. Distance Sensing

The HC-SR04 works by sending an ultrasonic pulse from the `Trig` pin and measuring how long it takes for the echo to return on the `Echo` pin. That time is converted to a distance using the speed of sound:

```
distance (cm) = (echo duration in microseconds × 0.034) / 2
```

The division by 2 accounts for the fact that the pulse travels to the object *and back*.

### 2. Presence Detection

On every loop iteration, the Arduino takes a fresh distance reading. If that distance falls within the configured `detectionRange` (default: 100 cm), the system considers a person "present" and immediately turns the light on, also recording the current timestamp (`lastDetectedTime`).

### 3. The 30-Second Grace Period

This is the core design decision of the project. A naive implementation would turn the light off the instant the sensor stops detecting a person. In practice, this causes annoying flicker — someone standing still near the edge of the detection range, or briefly turning away, would cause the light to blink on and off repeatedly.

Instead, every time a person *is* detected, the system updates `lastDetectedTime`. The light only turns off once **30 full seconds have passed since the last positive detection** — giving a smooth, forgiving experience that tolerates brief gaps in detection.

### 4. Sleep Mode

The Sleep Mode switch is checked at the very start of every loop iteration. If it's active, the function returns early after forcing the light off — completely bypassing the sensing and timing logic. This makes Sleep Mode a true override: no matter what the sensor sees, the light stays off.

---

## Software Architecture

The program is structured as a simple **finite state machine** with three effective states:

```
        ┌───────────┐
        │  SLEEP    │  (switch active — light forced OFF)
        └─────┬─────┘
              │ switch OFF
              ▼
        ┌───────────┐   person detected    ┌───────────┐
        │   OFF     │ ────────────────────▶│    ON     │
        └───────────┘                       └─────┬─────┘
              ▲                                    │
              │        30s with no detection        │
              └────────────────────────────────────┘
```

Rather than modeling this with an explicit `enum` (which would be a reasonable next step for a larger project), the current implementation tracks state implicitly through two variables: `lightState` (boolean) and `lastDetectedTime` (timestamp). This keeps the code compact and easy to follow for a project of this scope.

---

## Code Walkthrough

**Non-blocking timing with `millis()`**

The single most important technical choice in this project is using `millis()` instead of `delay()` for the 30-second wait:

```cpp
if (personPresent) {
  lastDetectedTime = millis();
  setLight(true);
} else if (lightState && (millis() - lastDetectedTime >= offDelay)) {
  setLight(false);
}
```

`millis()` returns the number of milliseconds since the Arduino started running. By comparing the *difference* between the current time and the last detection time against `offDelay` (30000 ms), the program can "wait" without ever calling `delay()` — meaning the ultrasonic sensor keeps taking readings every loop cycle instead of freezing for 30 seconds.

**Distance measurement**

```cpp
long getDistanceCM() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH, 30000);
  if (duration == 0) return -1;

  return duration * 0.034 / 2;
}
```

The `pulseIn()` timeout (30 ms) prevents the program from hanging indefinitely if no echo is received — for example, if the sensor is pointed at empty space beyond its effective range.

**Avoiding redundant writes**

```cpp
void setLight(bool state) {
  if (state != lightState) {
    digitalWrite(lightPin, state ? HIGH : LOW);
    lightState = state;
  }
}
```

This helper only writes to the pin when the state actually changes, rather than every single loop cycle — a small efficiency habit that also avoids relay chatter if a mechanical relay is used.

---

## Setup & Installation

1. Wire the components according to the [pin connection table](#circuit-diagram--pin-connections) above.
2. Install the Arduino IDE from [arduino.cc](https://www.arduino.cc/en/software) if you haven't already.
3. Open `smart_light_automation.ino` in the Arduino IDE.
4. Select your board (**Tools → Board → Arduino Uno**) and correct COM port.
5. Click **Upload**.
6. Open the Serial Monitor (optional) to observe behavior while testing.
7. Adjust `detectionRange` and `offDelay` constants near the top of the file to match your room size and desired sensitivity.

---

## Usage

- Walk toward the sensor — the light should turn on almost instantly.
- Walk away and wait — the light should stay on for 30 seconds before switching off.
- Flip the Sleep Mode switch — the light should turn off immediately and stay off regardless of movement, until the switch is flipped back.

---

## Challenges & Solutions

**Problem: Light flickering when standing near the detection boundary.**
Early testing without the grace period caused the light to rapidly toggle on/off as the ultrasonic reading fluctuated slightly around the threshold distance. Adding the 30-second `millis()`-based delay resolved this by requiring sustained absence before switching off.

**Problem: `delay()` made the sensor unresponsive.**
An early version used `delay(30000)` for the off-timer, which froze the entire program for 30 seconds — meaning a person walking back into range during that window wasn't detected at all. Switching to `millis()`-based non-blocking timing fixed this completely.

**Problem: Occasional invalid sensor readings.**
The HC-SR04 sometimes returns `0` when no echo is detected (e.g., soft/angled surfaces absorbing the pulse). The code checks for this and treats it as "no detection" rather than misinterpreting it as a very large or very small distance.

---

## What I Learned

- Why `millis()`-based timing is preferred over `delay()` in any project that needs to stay responsive
- How ultrasonic distance sensing actually works at the pulse-timing level
- How to design simple embedded systems using state-machine thinking, even without a formal state enum
- Practical debouncing/hysteresis concepts — not just for buttons, but for sensor-driven automation

---

## Future Improvements

- Replace the physical Sleep Mode switch with a Real-Time Clock (RTC) module for automatic time-based scheduling
- Add a PIR sensor alongside the ultrasonic sensor for improved detection reliability
- Add Wi-Fi (via ESP32) for remote monitoring and control through a mobile app
- Log detection events with timestamps for basic usage analytics
- Replace the relay with a solid-state relay (SSR) for silent, faster switching

---

## License

This project is open for personal and educational use. Feel free to fork, modify, and build on it.
