---
title: Architecture
---

## Overview

The firmware is written in C on top of **ESP-IDF** (FreeRTOS), with no Arduino layer. It is organized as a set of ESP-IDF components, each owning one concern (the charging state machine, the peripherals, networking, the protocol servers, scripting, and so on). The application wires these together at boot and then runs a small periodic control loop.

A central idea of the project is the **hardware abstraction**: the firmware binary contains no GPIO numbers, ADC channels or board-specific wiring. All of that lives in a `board.yaml` file on a dedicated flash partition and is read at boot. The same binary therefore runs on any board built around a given ESP32 family chip. See [Board config schema](Board-config-schema.md) and the [build examples](../10-hardware/build-examples.md).

## Components

The most relevant components are:

| Component | Responsibility |
| ------------ | -------------- |
| `evse` | The charging [state machine](state-machine.md) and the high-level charging settings |
| `peripherals` | Hardware drivers: control pilot, proximity, AC relay, socket lock, energy meter, RCM, temperature, LEDs, auxiliary IO |
| `config` | Loads and validates `board.yaml`, exposes the board configuration to the rest of the firmware |
| `network` | Wi-Fi station / access point, network services |
| `protocols` | Web server, REST API, WebDAV, OTA |
| `modbus` | [Modbus](Modbus.md) server (RTU and TCP) |
| `serial` | Serial port manager, shared by Modbus RTU, [Nextion](Nextion.md), [AT commands](AT-Commands.md) and scripting |
| `script` | [Lua](Lua.md) scripting engine |
| `logger` | Runtime logging |

The peripheral drivers each check the board configuration and simply do nothing when the corresponding hardware is not declared. For example, the socket lock driver only starts its task when a socket lock is configured, and the energy meter falls back to a calculated value when no current sensors are present.

## Boot sequence

At start-up the application performs, in order:

1. **Logging** is initialized.
2. **OTA rollback check.** If the running image is a freshly flashed OTA update pending verification, a diagnostic runs; on success the image is marked valid, otherwise the device rolls back to the previous image. See [OTA and recovery](#ota-and-recovery).
3. **NVS** (non-volatile storage) is mounted; if it is corrupt or from an incompatible version it is erased and re-created.
4. **File system.** A LittleFS partition is mounted at `/usr`, formatting it if the mount fails. This holds the web assets and scripts.
5. **Board configuration** is loaded from its partition (with a safety fallback, see [Boot-loop protection](#boot-loop-protection)).
6. Subsystems are initialized: network, peripherals, Modbus, serial, protocols, the EVSE state machine, the button, and scripting.

After initialization the application enters its main loop.

## The main loop

A single task runs the control loop approximately every **50&nbsp;ms**. Each pass:

- runs one iteration of the [state machine](state-machine.md),
- services the **button** (short press / long press),
- processes **Wi-Fi** connection state and updates the Wi-Fi LED,
- updates the **charging** and **error** LEDs from the current state.

Time-critical and blocking work runs in its own FreeRTOS tasks rather than in this loop &ndash; for example the socket lock actuation sequence, the energy meter sampling, the network stack and the protocol servers. The control loop stays short so that pilot measurement and state evaluation happen at a steady cadence.

## Configuration and persistence

There are two distinct kinds of configuration:

- **Board configuration** (`board.yaml`) describes the *hardware*: which GPIOs and ADC channels are wired to what, and the calibration thresholds. It is read-only at runtime and lives on its own partition so it survives firmware updates. It can be deployed with the [web installer](../installer.md) or over WebDAV.
- **Settings** describe *operation*: maximum and default charging current, energy-meter mode and AC voltage, socket-lock timing, authorization requirement, temperature threshold, limits, and so on. These are stored in **NVS** and are changed from the web UI, the REST API, Modbus, scripts or AT commands.

The split means a firmware update or a settings reset never touches the hardware description, and moving the firmware to a different board is only a matter of providing a different `board.yaml`.

## Button

The configured button has two functions:

- **Short press** &ndash; start the Wi-Fi access point (used for initial setup or to rejoin a network). The AP stays up for about 60&nbsp;seconds waiting for a client.
- **Long press** (about 10&nbsp;seconds) &ndash; mark the charger unavailable and perform a **factory reset**, erasing NVS and rebooting. The hardware description in `board.yaml` is not affected.

## OTA and recovery

Firmware is updated over the air. A new image is written to the inactive application partition and the device boots into it on the next restart. The freshly booted image must pass a diagnostic before it is marked valid; if it crashes or fails the check, the bootloader rolls back to the previously working image. OTA channels (stable, and optionally testing / bleeding-edge) are declared in `board.yaml`.

## Boot-loop protection

To avoid getting stuck in a crash loop caused by a bad hardware description, the firmware counts consecutive abnormal restarts in RTC memory (memory that survives a reset but not a power cycle). If too many panics happen in a row, the board configuration is loaded in a **safe/default** mode so the device can still come up far enough to be reconfigured or recovered, rather than panicking again on the same configuration.

## See also

- [State machine](state-machine.md) &ndash; the charging control sequence.
- [Charging control](charging-control.md) &ndash; the operational settings the state machine consumes.
- [Board config schema](Board-config-schema.md) &ndash; the structure of `board.yaml`.
- [Modbus](Modbus.md), [Lua](Lua.md), [Nextion](Nextion.md), [AT commands](AT-Commands.md), [REST API](rest-api.md) &ndash; external interfaces.
