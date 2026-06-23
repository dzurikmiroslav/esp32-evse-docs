---
title: Control Pilot
---

## Overview

The **Control Pilot** (CP) is the signalling line defined by IEC 61851-1 / SAE J1772 that the charger and the vehicle use to coordinate charging. The charger drives CP and reads it back; the [state machine](../20-software/state-machine.md) is built entirely on top of what CP reports.

The charger applies a **±12&nbsp;V, 1&nbsp;kHz square wave** to CP, referenced to protective earth (PE). On the vehicle side a diode and a switchable resistor load the positive half of the wave, pulling the positive peak down to defined levels. The charger measures the positive peak to learn the vehicle's request, and the duty cycle of the square wave tells the vehicle how much current it may draw.

| Positive peak | State | Meaning |
| ------------- | ----- | ------- |
| 12&nbsp;V | A | No vehicle connected |
| 9&nbsp;V  | B | Vehicle connected |
| 6&nbsp;V  | C | Vehicle requests charging |
| 3&nbsp;V  | D | Vehicle requests charging with ventilation |
| below 3&nbsp;V | &ndash; | Treated as a fault by the firmware |

## Generating the pilot

The firmware produces the square wave with the ESP32 LEDC (PWM) peripheral on the configured pilot GPIO, at **1&nbsp;kHz** with **10-bit** duty resolution. That logic-level signal is converted by an op-amp stage on the board into the ±12&nbsp;V swing presented on CP.

The pilot can be driven three ways, and the state machine switches between them:

- **Steady +12&nbsp;V** &ndash; the PWM is stopped high. Used in states A, B1, C1 and D1 (charger present but not energizing).
- **Steady &minus;12&nbsp;V** &ndash; the PWM is stopped low. Used in states E and F (error / unavailable).
- **PWM** &ndash; the duty cycle advertises the available current. Used in states B2, C2 and D2.

### Advertising current

The duty cycle encodes the current the charger offers, following the J1772 relationship:

- **6&nbsp;A to 51&nbsp;A:** *current&nbsp;=&nbsp;duty&nbsp;%&nbsp;×&nbsp;0.6* (so duty&nbsp;%&nbsp;=&nbsp;current&nbsp;/&nbsp;0.6). For example 16&nbsp;A is about 26.7&nbsp;% duty.
- **51&nbsp;A to 80&nbsp;A:** *current&nbsp;=&nbsp;(duty&nbsp;%&nbsp;&minus;&nbsp;64)&nbsp;×&nbsp;2.5*.

The advertised value is the working charging current capped by the cable rating; see [Charging control](../20-software/charging-control.md#charging-current).

## Sensing the pilot

CP is read back through a divider-with-offset network that maps the ±12&nbsp;V line into the ESP32 ADC's input range. Each measurement samples the ADC many times across a single 1&nbsp;kHz period and keeps the extremes:

- The **highest** samples give the positive peak, which selects the state (A/B/C/D, or a fault if it falls below the level for D).
- The **lowest** samples are checked against the &minus;12&nbsp;V level to confirm the negative half is present.

Because the mapping from CP volts to ADC millivolts depends on the exact resistor values of each board, the decision thresholds are not hard-coded. They are listed per board in `board.yaml` and set during [Control Pilot calibration](CP-calibration.md).

### Diode short detection

In a healthy connection the diode on the vehicle side clamps the negative half of the wave, so when the PWM is running the charger still sees a proper &minus;12&nbsp;V on the low half. If the PWM is running but the negative half does **not** reach &minus;12&nbsp;V, the vehicle diode is missing or shorted; the firmware raises a `DIODE_SHORT` error and refuses to energize. This is an important safety check &ndash; without the diode the state encoding is no longer trustworthy.

## Configuration

The pilot is declared in `board.yaml`:

```yaml
pilot:
  gpio: 33            # PWM output GPIO (to the op-amp stage)
  adcChannel: 3       # ADC1 channel sensing CP
  levels: [2405, 2099, 1792, 1484, 728]  # thresholds for 12, 9, 6, 3, -12 V in mV
```

The five `levels` are the measured ADC voltages (in mV) for the 12&nbsp;V, 9&nbsp;V, 6&nbsp;V, 3&nbsp;V and &minus;12&nbsp;V points. See [Control Pilot calibration](CP-calibration.md) for how to obtain them for your board.

!!! danger
    The Control Pilot circuit and its calibration are safety-critical. Incorrect thresholds can cause the charger to misread the vehicle state &ndash; for example energizing when it should not, or failing to detect a diode fault. Calibrate against a known-good reference and verify every state before connecting a vehicle.

## See also

- [Control Pilot calibration](CP-calibration.md) &ndash; obtaining the `levels` values.
- [State machine](../20-software/state-machine.md) &ndash; how CP readings drive charging.
- [Proximity Pilot](proximity-pilot.md) &ndash; the companion line that codes cable current.
