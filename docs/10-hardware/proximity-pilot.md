---
title: Proximity Pilot
---

## Overview

The **Proximity Pilot** (PP) is the second signalling contact of a Type&nbsp;2 connector. It tells the charger the **current-carrying capacity of the attached cable**, so the charger never advertises more current than the cable can safely carry. It is only relevant when the charger has a **socket outlet** (a detachable cable); a charger with a permanently attached cable knows its own cable and does not use PP.

A resistor built into the cable's plug, between PP and protective earth, codes the rating. The charger reads the resulting voltage on PP and maps it to a current.

| Coded rating | Typical use |
| ------------ | ----------- |
| 13&nbsp;A | Low-power (granny) chargers |
| 20&nbsp;A | Standard 16 A/20 A cables |
| 32&nbsp;A | Common Type&nbsp;2 cable |

## How it is used

The PP contact is read through a divider into the ESP32 ADC. The measured voltage is compared against three thresholds to classify the cable as 13&nbsp;A, 20&nbsp;A or 32&nbsp;A; anything below the 32&nbsp;A threshold is taken as 63&nbsp;A.

The reading is taken when the vehicle is plugged in, as the session begins (state B1). The resulting rating becomes an upper bound on the advertised [Control Pilot](control-pilot.md) current: the charger advertises the working charging current **capped by the cable rating**. So even if the maximum charging current is set higher, a 16&nbsp;A cable limits the session to what that cable supports.

Socket-outlet mode requires a Proximity Pilot to be present on the board. With a fixed cable, PP is not needed and the maximum current is governed only by the configured maximum charging current.

## Configuration

PP is declared in `board.yaml`:

```yaml
proximity:
  adcChannel: 2          # ADC1 channel sensing PP
  levels: [1650, 820, 430]   # thresholds for 13 A, 20 A, 32 A in mV
```

The three `levels` are the measured ADC voltages (in mV) at the 13&nbsp;A, 20&nbsp;A and 32&nbsp;A boundaries; readings under the 32&nbsp;A threshold are classified as 63&nbsp;A. As with the control pilot, the exact figures depend on the board's divider and should be measured for your design.

If you're building an EVSE without a socket-outlet (having the cable directly attached) omit this configuration from `board.yaml`.

## See also

- [Control Pilot](control-pilot.md) &ndash; the line that advertises current to the vehicle.
- [Socket lock](socket-lock.md) &ndash; mechanically retaining the plug in a socket outlet.
- [Charging control](../20-software/charging-control.md#charging-current) &ndash; how the cable rating caps the advertised current.
