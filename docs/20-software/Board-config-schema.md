---
title: Board configuration schema
---

# Board configuration schema

The firmware ships as **one binary per ESP32 family** and contains no GPIO
numbers, ADC channels or board-specific wiring. Everything that describes a
particular board lives in a single YAML file, `board.yaml`, read at boot. The
same firmware therefore runs on any board built around a given chip — you only
write a config file. See [Architecture](architecture.md) for the bigger picture.

This page documents the current schema, `board-config-schema-1.json`
([in the repo](https://github.com/dzurikmiroslav/esp32-evse/tree/master/board-config)).

!!! note
    The auto-generated reference that previously lived here described the older
    *flat* schema (top-level keys like `ledChargingGpio`, `buttonGpio`). The
    current schema is **nested** (`leds.charging.gpio`, `button.gpio`, …). The
    structure and field names below match the nested schema actually parsed by
    the firmware.

## Where the file lives

`board.yaml` is stored on a dedicated **LittleFS** partition labelled `usr`
(320 KB), mounted at `/usr`, so the full path on the device is
`/usr/board.yaml`.

- On first boot, or whenever the file is missing, the firmware writes a built-in
  default for the target chip (`board_esp32.yaml`, `board_esp32s2.yaml` or
  `board_esp32s3.yaml`) to that path.
- A **factory reset** (hold the button) renames the current file to
  `board_invalid.yaml` and regenerates the default on the next boot.
- The same partition is exposed over [WebDAV](rest-api.md#related-webdav-file-access),
  so you can edit the file in place at `dav://<device-ip>/dav/usr/board.yaml`.
  It also holds the [Lua](Lua.md) scripts under `/usr/lua/`.

!!! warning
    The config is parsed with a hard assertion: if `board.yaml` is **malformed
    or fails validation, the device aborts and reboots** rather than starting
    with a bad configuration. Validate before you save — keep a known-good copy,
    and use the schema in your editor (below) to catch mistakes early.

### Editor autocompletion

Both example files start with a language-server directive so editors with the
YAML extension give you completion and validation against the schema:

```yaml
# yaml-language-server: $schema=https://github.com/dzurikmiroslav/esp32-evse/raw/refs/heads/master/board-config/board-config-schema-1.json
```

## GPIO and ADC conventions

- All pin fields are **raw GPIO numbers** for the chip, not board silk-screen
  labels. Pick pins valid for the function on your chip (e.g. input-only pins
  cannot drive a relay).
- Every `adcChannel` / `adcChannels` field is an **ADC1 channel index**, not a
  GPIO number. Only ADC1 is usable while Wi-Fi is active. Map the channel to its
  pin from the Espressif datasheet for your chip. The control pilot and energy
  meter sensing must be on ADC1.

## Top-level structure

| Key | Type | Required | Purpose |
| --- | ---- | :------: | ------- |
| `deviceName` | string (≤ 32) | ✅ | Human-readable device name |
| `button` | object | ✅ | The multifunction button |
| `pilot` | object | ✅ | [Control pilot](../10-hardware/control-pilot.md) (PWM + sensing) |
| `acRelay` | object | ✅ | Mains contactor control |
| `ota` | object | ✅ | OTA update channels |
| `leds` | object | — | Status LEDs |
| `proximity` | object | — | [Proximity pilot](../10-hardware/proximity-pilot.md) sensing |
| `socketLock` | object | — | [Socket lock](../10-hardware/socket-lock.md) actuator |
| `rcm` | object | — | [Residual current monitor](../10-hardware/residual-current-monitor.md) |
| `energyMeter` | object | — | [Energy metering](../10-hardware/energy-metering.md) |
| `aux` | object | — | Auxiliary GPIO / ADC exposed to scripting |
| `serials` | array | — | Serial ports (UART / RS-485) |
| `onewire` | object | — | 1-Wire bus (temperature sensor) |

No properties beyond those listed in each object are allowed
(`additionalProperties: false` throughout) — a typo'd key is a validation error.
A peripheral that is simply omitted is treated as absent, and its driver does
nothing.

## Required sections

### `deviceName`

A string up to 32 characters, shown in the UI and via
[`GET /board-config`](rest-api.md#get-board-config).

### `button`

The single multifunction button (short press / long press / factory reset).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpio` | integer | ✅ | Button GPIO (active by board wiring) |

```yaml
button:
  gpio: 0
```

### `pilot`

Control pilot: a PWM output plus an ADC1 input that reads the CP voltage to
detect vehicle state. See [Control pilot](../10-hardware/control-pilot.md) and
[CP calibration](../10-hardware/CP-calibration.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpio` | integer | ✅ | PWM generation GPIO |
| `adcChannel` | integer | ✅ | ADC1 channel that measures CP |
| `levels` | integer[5] | ✅ | ADC thresholds in **mV** for the +12, +9, +6, +3 and −12 V CP levels, in that order |

```yaml
pilot:
  gpio: 27
  adcChannel: 0
  levels: [2405, 2099, 1792, 1484, 728]
```

The five `levels` are board-specific (they depend on the CP divider/clamp
network) and are what the [CP calibration](../10-hardware/CP-calibration.md)
procedure produces.

### `acRelay`

Controls the mains contactor.

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpios` | integer[1..2] | ✅ | Contactor control GPIOs |

The array length selects single- vs. split-phase switching:

- **One GPIO** — drives all phases together (L1–L3).
- **Two GPIOs** — the first drives L1, the second drives L2–L3, allowing the
  EVSE to switch between single-phase and three-phase charging.

```yaml
acRelay:
  gpios: [26]
```

### `ota`

Over-the-air update sources. Each channel points at a JSON descriptor that the
[firmware update endpoints](rest-api.md#firmware-ota) consult.

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `channels` | array[1..3] | ✅ | One to three OTA channels |

Each **channel**:

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `name` | string (≤ 16) | ✅ | Channel label shown in the UI |
| `path` | string | ✅ | URL of the channel's JSON descriptor |

```yaml
ota:
  channels:
    - name: stable
      path: https://dzurikmiroslav.github.io/esp32-evse/ota/stable/esp32.json
    # - name: testing
    #   path: https://dzurikmiroslav.github.io/esp32-evse/ota/testing/esp32.json
```

## Optional sections

### `leds`

Up to three status LEDs; each is an object with a single `gpio`. Any of the
three may be omitted.

| Sub-key | Meaning |
| ------- | ------- |
| `charging` | Charging-state LED |
| `error` | Error LED |
| `wifi` | Wi-Fi-state LED |

```yaml
leds:
  charging:
    gpio: 19
  error:
    gpio: 18
  wifi:
    gpio: 17
```

### `proximity`

Proximity pilot sensing — reads the cable's current rating resistor. Used in
socket-outlet mode. See [Proximity pilot](../10-hardware/proximity-pilot.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `adcChannel` | integer | ✅ | ADC1 channel measuring PP |
| `levels` | integer[3] | ✅ | ADC thresholds in **mV** for 13 A, 20 A and 32 A cables, in that order |

```yaml
proximity:
  adcChannel: 3
  levels: [1650, 820, 430]
```

### `socketLock`

Motorized socket lock actuator (Type-2 socket outlets). See
[Socket lock](../10-hardware/socket-lock.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpios` | integer[2] | ✅ | Drive GPIOs for lock coil A and B (direction) |
| `detectionGpio` | integer | ✅ | Lock-state feedback input |
| `detectionDelay` | integer | ✅ | Delay after actuating before reading state, in ms |
| `minBreakTime` | integer | ✅ | Minimum pause between repeated lock/unlock actions, in ms |

```yaml
socketLock:
  gpios: [20, 19]
  detectionGpio: 34
  detectionDelay: 1000
  minBreakTime: 1000
```

The operating time, retry count and detection polarity are **runtime** settings
([`/config/evse`](rest-api.md#getpost-configevse)), not part of `board.yaml`.

### `rcm`

Residual current monitor (6 mA DC / AC fault detection with self-test). See
[Residual current monitor](../10-hardware/residual-current-monitor.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpio` | integer | ✅ | Fault-sense input GPIO |
| `testGpio` | integer | ✅ | Self-test trigger output GPIO |

```yaml
rcm:
  gpio: 41
  testGpio: 26
```

### `energyMeter`

Integrated energy meter. `current` sensing is required; `voltage` sensing is
optional — without it the meter uses the configured nominal AC voltage. The
number of `adcChannels` (1 or 3) determines single- vs. three-phase measurement.
See [Energy metering](../10-hardware/energy-metering.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `current` | object | ✅ | Current sensing |
| `voltage` | object | — | Voltage sensing |

Each of `current` / `voltage`:

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `adcChannels` | integer[1..3] | ✅ | ADC1 channels for L1 (L2, L3). One item = single phase |
| `scale` | number | ✅ | Multiplier converting the raw measurement to A or V |

```yaml
energyMeter:
  current:
    adcChannels: [4, 5, 6]
    scale: 0.090909091
  voltage:
    adcChannels: [7, 8, 9]
    scale: 0.47
```

The active meter mode (`dummy` / `cur` / `cur_vlt`) and three-phase flag are
chosen at runtime via [`/config/evse`](rest-api.md#getpost-configevse), but are
constrained by what the hardware here declares.

### `aux`

Auxiliary IO surfaced to [Lua scripting](Lua.md) and reported by
[`GET /board-config`](rest-api.md#get-board-config). Each entry has a short
`name` (≤ 8 characters) used to reference it from scripts.

| Sub-key | Type | Max entries | Entry fields |
| ------- | ---- | :---------: | ------------ |
| `inputs` | array | 4 | `name`, `gpio` |
| `outputs` | array | 4 | `name`, `gpio` |
| `analogInputs` | array | 2 | `name`, `adcChannel` (ADC1) |

```yaml
aux:
  inputs:
    - name: IN1
      gpio: 11
  outputs:
    - name: OUT1
      gpio: 17
  analogInputs:
    - name: IN3
      adcChannel: 0
```

!!! note
    The schema text expresses these caps with `maxLength`, which JSON Schema
    only applies to strings, so a generic validator will not reject an
    over-long array. The firmware enforces the real limits (**4 inputs, 4
    outputs, 2 analog inputs**); extra entries are ignored or rejected at load.

### `serials`

Serial ports, in order. Each port can be plain UART or RS-485, and its
**function** (log, Modbus, Nextion, AT, script) is selected at runtime via
[`/config/serial`](rest-api.md#getpost-configserial). Shared by
[Modbus](Modbus.md) RTU, [Nextion](Nextion.md) and [AT commands](AT-Commands.md).

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `type` | string | ✅ | `uart` or `rs485` |
| `name` | string (≤ 16) | ✅ | Label shown in the UI |
| `rxdGpio` | integer | ✅ | RX GPIO |
| `txdGpio` | integer | ✅ | TX GPIO |
| `rtsGpio` | integer | rs485 only | Flow-control / direction GPIO — **required** when `type: rs485` |

```yaml
serials:
  - type: uart
    name: UART
    rxdGpio: 12
    txdGpio: 13
  - type: rs485
    name: RS-485
    rxdGpio: 40
    txdGpio: 38
    rtsGpio: 39
```

!!! note
    The schema allows up to 3 entries, but the real ceiling is the chip's UART
    count (`SOC_UART_NUM`): **3 on ESP32 / ESP32-S3, 2 on ESP32-S2**.
    Additionally, the UART used for the system console is reserved and forced to
    "no port" at boot, so one slot is effectively consumed by the console on a
    standard build.

### `onewire`

A 1-Wire bus, currently used for a temperature sensor.

| Field | Type | Required | Notes |
| ----- | ---- | :------: | ----- |
| `gpio` | integer | ✅ | 1-Wire data GPIO |
| `temperatureSensor` | boolean | ✅ | Whether a temperature sensor is present on the bus |

```yaml
onewire:
  gpio: 16
  temperatureSensor: true
```

## Complete examples

### Minimal single-phase board (ESP32-DevKitC)

LEDs, button, pilot, proximity, a single-phase current+voltage meter, three
serial ports, a 1-Wire temperature sensor and one OTA channel. No socket lock or
RCM (fixed-cable style).

```yaml
# yaml-language-server: $schema=https://github.com/dzurikmiroslav/esp32-evse/raw/refs/heads/master/board-config/board-config-schema-1.json

deviceName: ESP32-DevKitC EVSE

leds:
  charging:
    gpio: 19
  error:
    gpio: 18
  wifi:
    gpio: 17

button:
  gpio: 0

pilot:
  gpio: 27
  adcChannel: 0
  levels: [2405, 2099, 1792, 1484, 728]

proximity:
  adcChannel: 3
  levels: [1650, 820, 430]

acRelay:
  gpios: [26]

energyMeter:
  current:
    adcChannels: [7]
    scale: 0.090909091
  voltage:
    adcChannels: [6]
    scale: 0.47

serials:
  - type: uart
    name: UART via USB
    rxdGpio: 3
    txdGpio: 1
  - type: rs485
    name: RS485
    rxdGpio: 32
    txdGpio: 25
    rtsGpio: 33
  - type: uart
    name: UART
    rxdGpio: 2
    txdGpio: 15

onewire:
  gpio: 16
  temperatureSensor: true

ota:
  channels:
    - name: stable
      path: https://dzurikmiroslav.github.io/esp32-evse/ota/stable/esp32.json
```

### Full three-phase board (ESP32-S2-DA)

Adds a socket lock, RCM, three-phase current+voltage metering and auxiliary IO.

```yaml
# yaml-language-server: $schema=https://github.com/dzurikmiroslav/esp32-evse/raw/refs/heads/master/board-config/board-config-schema-1.json

deviceName: ESP32-S2-DA EVSE

leds:
  charging:
    gpio: 36
  error:
    gpio: 37
  wifi:
    gpio: 35

button:
  gpio: 0

pilot:
  gpio: 33
  adcChannel: 3
  levels: [2405, 2099, 1792, 1484, 728]

proximity:
  adcChannel: 2
  levels: [1650, 820, 430]

acRelay:
  gpios: [21]

socketLock:
  gpios: [20, 19]
  detectionGpio: 34
  detectionDelay: 1000
  minBreakTime: 1000

rcm:
  gpio: 41
  testGpio: 26

energyMeter:
  current:
    adcChannels: [4, 5, 6]
    scale: 0.090909091
  voltage:
    adcChannels: [7, 8, 9]
    scale: 0.47

aux:
  inputs:
    - name: IN1
      gpio: 11
    - name: IN2
      gpio: 2
  outputs:
    - name: OUT1
      gpio: 17
    - name: OUT2
      gpio: 16
    - name: OUT3
      gpio: 15
    - name: OUT4
      gpio: 14
  analogInputs:
    - name: IN3
      adcChannel: 0

serials:
  - type: uart
    name: UART
    rxdGpio: 12
    txdGpio: 13
  - type: rs485
    name: RS-485
    rxdGpio: 40
    txdGpio: 38
    rtsGpio: 39

onewire:
  gpio: 42
  temperatureSensor: true

ota:
  channels:
    - name: stable
      path: https://dzurikmiroslav.github.io/esp32-evse/ota/stable/esp32s2.json
```

## See also

- [Build examples](../10-hardware/build-examples.md) — reference hardware designs
- [Architecture](architecture.md) — how the config drives the firmware
- [REST API: `GET /board-config`](rest-api.md#get-board-config) — read the parsed result at runtime
