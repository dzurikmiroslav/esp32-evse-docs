---
title: ESP32-S2 EVSE DIY Alpha
---

## Overview

A board based on ESP32-S2 you can assemble yourself is ESP32-S2 EVSE DIY Alpha. Has onboard three phase energy meter, RS485, UART, 1WIRE, RCM, socket lock.

![ESP32-S2-DA](/images/esp32s2da.jpg)

You can order the PCB from the [EasyEDA project](https://oshwlab.com/dzurik.miroslav/esp32s2-diy-evse) page, even assembled.

![Pinout](/images/esp32s2da-terminals.png)

The dimensions of the board are 150x122 mm, designed to fit into the UM122 enclosure.

### Buttons

| Name  | Usage                                                | Note             |
| ----- | ---------------------------------------------------- | ---------------- |
| BOOT  | Activate WiFi AP, during restart enter flashing mode | GPIO0 pin on ESP |
| RESET | Restart controller                                   | EN pin ESP       |

### LEDs

| Name | Usage    | Color | Description                                                                                                                                   |
| ---- | -------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| WIFI | WiFi     | Blue  | on - connected to WiFi <br/> flashing (on/off 100/900ms) - Started WiFi AP <br/> flashing (on/off 1900/100ms) - WiFi AP has client connection |
| CHR  | Charging | Green | flashing - vehicle connected but not charging <br/> on - vehicle charging                                                                     |
| ERR  | Error    | Red   | on - is in error state (E) <br/> flashing - is in not available state (F)                                                                     |

### Terminals

| Name   | Usage                                  | Note                                     |
| ------ | -------------------------------------- | ---------------------------------------- |
| B      | Detection of the locking actuator      |                                          |
| R      | Control of the locking actuator        |                                          |
| W      | Control of the locking actuator        |                                          |
| CP     | Control Pilot                          |                                          |
| PP     | Proximity Plug                         |                                          |
| GND    | Ground                                 | System ground                            |
| 12V    | 12V                                    |                                          |
| OUT1   | Low side output 1                      |                                          |
| OUT2   | Low side output 2                      |                                          |
| OUT3   | Low side output 3                      |                                          |
| OUT4   | High side output 1                     |                                          |
| IN1    | Digital input 1                        |                                          |
| IN2    | Digital input 2                        |                                          |
| IN3    | Analog input                           |                                          |
| GND    | Ground                                 | System ground                            |
| DATA   | 1-Wire bus                             | For external temperature sensor DS18B20  |
| 5V     | 5V power supply                        |                                          |
| 12V    | 12V power supply                       |                                          |
| A+     | RS-485 A+ pin                          | RS-485 bus with 120Ω terminator resistor |
| B-     | RS-485 B- pin                          | RS-485 bus with 120Ω terminator resistor |
| UART   | UART serial bus (with 5V power supply) | RX and TX at 3.3V level directly on ESP  |
| AR_RLY | Relay output for AC contactor          |                                          |
| PE     | Protective Earth                       | Connected to system ground               |
| N      | Neutral                                | Neutral conductor, power grid            |
| L      | Line                                   | Phase input, for powering the board      |
| L1     | Energy meter L1 voltage input          |                                          |
| L2     | Energy meter L2 voltage input          |                                          |
| L3     | Energy meter L3 voltage input          |                                          |
| CT1    | Energy meter L1 current transformer    |                                          |
| CT2    | Energy meter L2 current transformer    |                                          |
| CT3    | Energy meter L3 current transformer    |                                          |
| RCM    | Residual current monitor               | RCM14-01, RCM14-03                       |

## Socket lock

This board can drive a motorized **socket lock** actuator, used in socket-outlet mode to retain the plug for the duration of a charging session. Three terminals are dedicated to it:

| Terminal | Role |
| -------- | ---- |
| R | Actuator control line A |
| W | Actuator control line B |
| B | Position detection input |

**R** and **W** are the two control lines of an H-bridge that drives the actuator motor. To **lock**, the board energizes the pair in one polarity for the operating time; to **unlock**, it energizes them in the opposite polarity. After each move both lines are released, so the motor is only powered for the brief operating window rather than held.

**B** is the detection input. To read the current position the board drives R and W to the same level &ndash; the motor sees no voltage difference and does not move &ndash; and reads the feedback contact on B after a short delay. A *detection polarity* setting selects whether the locked position reads high or low, so the same firmware works with actuators of either contact sense.

On this board the detection is read **1000&nbsp;ms** after a move, and the minimum break time between operations is **1000&nbsp;ms** (from its `board.yaml`). The operating time, break time, retry count and detection polarity are adjustable in settings. The lock engages automatically when a vehicle is plugged in and releases when it is unplugged or the charger faults; charging will not begin until the lock has confirmed the locked position, and a lock or unlock failure puts the charger into the error state.

The corresponding `board.yaml` block is:

```yaml
socketLock:
  gpios: [20, 19]      # control lines on terminals R and W
  detectionGpio: 34    # detection input on terminal B
  detectionDelay: 1000
  minBreakTime: 1000
```

For the full description of the locking sequence, detection, retries and fault handling, see [Socket lock](socket-lock.md). The wiring of a socket outlet with its actuator is shown in [With socket outlet](#with-socket-outlet) below.

## Connection examples

L1, L2, L3 connections onboard charging voltage meter, they are optional. Use 30mA fuses to connect them to the lines.

CT1, CT2, CT3 connections for current transformers measuring charging current, they are optional. Can be used any with ratio 2000/1 (for example: DL-CT08CL5 or SCT-013-000). 

If you don't want to have an onboard meter, set `Energy meter` `Mode` to `Dummy`, power will be calculated from actual charging current slider and the `AC voltage` value in the settings.

When you only have current transformers, set `Energy meter` `Mode` to `Current sensing`, power will be calculated from measured current and the `AC voltage` value in the settings.

When you have current transformers and voltages connected, set `Energy meter Mode` to `Current and voltage sensing`, power will be calculated from the real measured current and voltage.

Remember to check `Energy meter` checkbox `Three phases` in case your installation is relying on a three-phase connection.

`Residual Current Monitor` (RCM) is optional, if is not available it must be disabled in settings.

### With socket outlet

![Wiring socket outlet](/images/esp32s2da-wiring-lock.png)

Set `Max charging current` to the value of the circuit breaker that protects the EVSE.

### With fixed cable

![Wiring fixed cable](/images/esp32s2da-wiring-cable.png)

Set `Max charging current` to the lower value of the circuit breaker that protects the EVSE or cable maximum current.
