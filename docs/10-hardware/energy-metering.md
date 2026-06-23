---
title: Energy metering
---

## Overview

The firmware tracks **power**, **session energy** and per-phase **voltage and current** so it can show live figures, enforce the consumption and under-power [limits](../20-software/charging-control.md#charging-limits), and report over Modbus and the API. How those figures are obtained depends on what sensing hardware the board has, selected by the **energy meter mode**.

| Mode | Sensing used | Power is&hellip; |
| ---- | ------------ | --------------- |
| Dummy | none | calculated from the advertised charging current and a configured AC voltage |
| Current sensing | current transformers | calculated from measured current and a configured AC voltage |
| Current and voltage sensing | current transformers + voltage taps | the real product of measured current and measured voltage |

A board can only use a mode its hardware supports: current sensing needs current transformers, and current-and-voltage sensing additionally needs the voltage inputs.

## Modes in detail

### Dummy

With no measurement hardware, the meter assumes the vehicle draws exactly the current the charger advertises, and multiplies it by a configured **AC voltage** value. This gives a reasonable energy estimate for limits and display on a minimal board, but it cannot see the vehicle's actual draw.

### Current sensing

Current transformers (CTs) on the supply conductors measure the real charging current. Power and energy are computed from that current and the configured AC voltage. Any CT with a 2000:1 ratio is suitable (for example DL-CT08CL5 or SCT-013-000). This tracks the vehicle's actual draw, while still assuming a fixed line voltage.

### Current and voltage sensing

With both CTs and voltage taps, the meter measures current and voltage together and reports true power. This is the most accurate mode and the one to use when the board has voltage sensing fitted.

## Single and three phase

Set the **three phases** option to match the installation. On a three-phase supply with three-phase sensing, all three phases are measured; on a single-phase supply only one is used. The reported per-phase voltages and currents reflect whichever phases are active.

## AC voltage setting

In the dummy and current-sensing modes there is no measured voltage, so a configured **AC voltage** value (default 250&nbsp;V, adjustable) is used in the power calculation. In current-and-voltage mode this setting is not used because voltage is measured directly.

## Configuration

Current and voltage inputs are declared in `board.yaml` with their ADC channels and a scale factor, for example:

```yaml
energyMeter:
  current:
    adcChannels: [4, 5, 6]
    scale: 0.090909091
  voltage:
    adcChannels: [7, 8, 9]
    scale: 0.47
```

The presence of these blocks determines which modes the board can offer. The mode itself, the three-phase flag and the AC voltage are runtime settings stored in NVS.

## See also

- [Charging control](../20-software/charging-control.md#charging-limits) &ndash; the consumption and under-power limits that consume these figures.
- [ESP32-S2 EVSE DIY Alpha](esp32s2-evse-d-a.md#connection-examples) &ndash; wiring CTs and voltage inputs on a real board.
- [Modbus reference](../20-software/Modbus.md) &ndash; reading power, energy, voltage and current.
