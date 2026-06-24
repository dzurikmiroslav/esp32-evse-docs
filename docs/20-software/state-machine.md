---
title: State machine
---

## Overview

The core of the firmware is a state machine that implements the control sequence defined by **IEC 61851-1** / **SAE J1772**. It decides when to offer energy to the vehicle, when to close the AC contactor, and when to stop because of an error or a configured limit.

The vehicle and the charger talk over the **Control Pilot** (CP). The charger applies a ±12&nbsp;V signal to CP; the vehicle answers by switching resistors (through a diode) between CP and protective earth, which pulls the positive half of the CP voltage down to defined levels. The charger reads those levels back and uses them to drive the state machine. See [Control Pilot](../10-hardware/control-pilot.md) for the hardware side and [Control Pilot calibration](../10-hardware/CP-calibration.md) for how the voltage thresholds are set.

The standard defines states **A** to **F**. This firmware splits the active states **B**, **C** and **D** into two sub-states, **1** and **2**, to track whether the charger is currently energizing the vehicle.

## The 1 and 2 sub-states

The digit after the letter encodes what the charger is doing, not what the vehicle is asking for:

- **`1`** &ndash; the charger is **not** offering energy. The AC contactor is open and the Control Pilot is held at a steady **+12&nbsp;V** (PWM suppressed).
- **`2`** &ndash; the charger **is** offering energy. The Control Pilot runs a **1&nbsp;kHz PWM** that advertises the available current, and (in C2/D2) the AC contactor is closed.

So `B1` means *vehicle connected, charging not yet permitted*, while `B2` means *vehicle connected, charger ready and advertising current*. `C1` means *vehicle is asking to charge but the charger has paused*, while `C2` means *charging*.

## States

| State | CP voltage (vehicle) | Pilot output | AC contactor | Meaning |
| ----- | -------------------- | ------------ | ------------ | ------- |
| A     | 12&nbsp;V            | +12&nbsp;V steady | open    | No vehicle connected |
| B1    | 9&nbsp;V             | +12&nbsp;V steady | open    | Vehicle connected, charging not permitted yet |
| B2    | 9&nbsp;V             | PWM          | open         | Vehicle connected, charger ready, advertising current |
| C1    | 6&nbsp;V             | +12&nbsp;V steady | open*   | Vehicle requests charging, charger paused |
| C2    | 6&nbsp;V             | PWM          | closed       | Charging |
| D1    | 3&nbsp;V             | +12&nbsp;V steady | open*   | Vehicle requests charging **with ventilation**, charger paused |
| D2    | 3&nbsp;V             | PWM          | closed       | Charging, ventilation required |
| E     | &mdash;              | &minus;12&nbsp;V steady | open | Error |
| F     | &mdash;              | &minus;12&nbsp;V steady | open | Charger not available |

\* When entering C1/D1 from a charging state the contactor is not opened immediately, see [Stopping charging gracefully](#stopping-charging-gracefully).

States **C** and **D** differ only in that **D** indicates the vehicle requests *ventilated charging* (according to the standard, outdoors chargers should continue charging; indoors (like garage) chargers should trigger a ventilation system of the area and only continue charging when that works). The firmware treats them identically except for the state label; whether **D** charging is actually allowed is a function of the installation, not the firmware.

!!! note
    The firmware supports Lua scripting. With the help of a script garage ventilation can be activated in state **D** (using an AUX GPIO output).

!!! note
    Internally the firmware never stores `E` as a state. It keeps the "normal" state separately and reports `E` whenever any error bit is set. When all error bits are cleared the machine resumes from state **A**.

## The control loop

![State Machine](/images/state-machine-light.png#only-light)
![State Machine](/images/state-machine-dark.png#only-dark)

The state machine is evaluated periodically from the main loop, roughly every **50&nbsp;ms**. Each pass does the following:

1. **Measure the Control Pilot.** The pilot is sampled many times across one 1&nbsp;kHz period; the highest readings give the positive level (which selects A/B/C/D) and the lowest readings tell whether the negative half reaches &minus;12&nbsp;V.
2. **Check for a diode short.** If the PWM is running but the negative half does not reach &minus;12&nbsp;V, the vehicle diode is missing or shorted and the `DIODE_SHORT` error is raised.
3. **Check other error conditions** &ndash; socket lock faults, residual current, over-temperature (see [Errors](#errors)).
4. **Update the limit flags** &ndash; consumption, charging time and under-power (see [Charging limits](charging-control.md#charging-limits)).
5. **Run the transition** for the current state based on the measured CP level.
6. **Apply the new state** if it changed: set the pilot output, open or close the contactor, drive the socket lock, start or stop the energy meter session.

## Transitions

Transitions are driven by the positive CP level. Whether an active request results in a `1` (paused) or `2` (energizing) sub-state is decided by an internal check, *charging allowed*, evaluated on every transition.

**Charging is allowed** only when **all** of these hold:

- charging is **enabled** (see [Enabled and available](charging-control.md#gates-authorization-enabled-available)),
- the charger is **available**,
- the session is **authorized** (always true unless authorization is required),
- **no limit** has been reached,
- if a socket lock is used, it has settled to the **idle** (successfully locked) state.

The CP level then maps to the target as follows:

| Current state | CP 12&nbsp;V | CP 9&nbsp;V | CP 6&nbsp;V | CP 3&nbsp;V |
| ------------- | ------------ | ----------- | ----------- | ----------- |
| A             | A            | B1          | &mdash;     | &mdash;     |
| B1 / B2       | A            | B2 / B1     | C2 / C1     | &mdash;     |
| C1 / C2       | A            | B2 / B1     | C2 / C1     | D2 / D1     |
| D1 / D2       | &mdash;      | &mdash;     | C2 / C1     | D2 / D1     |

Where two targets are shown (e.g. `C2 / C1`), the first is taken when charging is allowed and the second when it is not. A CP level that is not valid for the current state raises a `PILOT_FAULT` error. Because of this, a clean disconnect is expected to pass through the normal sequence (D&nbsp;&rarr;&nbsp;C&nbsp;&rarr;&nbsp;B&nbsp;&rarr;&nbsp;A); an abrupt jump out of state **D** is treated as a pilot fault, which auto-clears.

This design is also how charging is paused without any explicit "stop" transition: if a limit is reached or the charger is disabled while in `C2`, *charging allowed* becomes false, and on the next CP reading the state moves from `C2` to `C1`.

### Entering a session (state B1)

State **B1** is the entry point of a charging session (a *session* is any state from B1 to D2). On the first entry to B1 the firmware:

- engages the **socket lock** (if fitted and socket-outlet mode is on),
- runs the **residual current monitor self-test** (if fitted),
- reads the **cable current rating** from the Proximity Pilot (in socket-outlet mode), see [Proximity Pilot](../10-hardware/proximity-pilot.md),
- starts the **energy meter session**.

The session ends &ndash; socket lock released, energy meter session stopped &ndash; when the machine returns to **A**, **E** or **F**.

### Stopping charging gracefully

When the machine leaves a charging state (`C2`/`D2`) for the paused state (`C1`/`D1`), it does **not** open the contactor immediately. Instead it raises the Control Pilot to a steady +12&nbsp;V and arms a **6&nbsp;second** timer. Raising the pilot signals the vehicle to wind down its current draw; a well-behaved vehicle then drops to state B (9&nbsp;V), at which point the contactor is opened with little or no current flowing. If the vehicle has not responded when the 6&nbsp;second timer expires, the contactor is forced open. This reduces contactor wear and arcing compared with opening under load.

## Authorization

By default a session starts automatically. When **authorization is required**, a session that reaches B1/C1/D1 stays unauthorized until it is granted, and *charging allowed* stays false (so the charger advertises nothing and the contactor stays open).

Authorization is granted by an external action &ndash; the web UI, the REST API, Modbus, a Lua script, or an AT command. A granted authorization is valid for **60&nbsp;seconds**; if the vehicle is not in a session state within that window the grant lapses. Authorization is also reset whenever the vehicle disconnects, so every plug-in requires a fresh authorization. While a session is waiting, the firmware exposes a *pending authorization* flag.

## Charging limits

Three optional limits can stop an in-progress session. Each is checked every loop and, when reached, makes *charging allowed* false so the machine drops to the paused sub-state:

- **Consumption limit** (Wh) &ndash; stop after a given amount of energy has been delivered.
- **Charging time limit** (s) &ndash; stop after a given session duration.
- **Under-power limit** (W) &ndash; stop when power demand decreases below a threshold for **60&nbsp;seconds** continuously.

Limits and the values that feed them are described in [Charging control](charging-control.md).

## Enabled and available

Two independent flags, settable from external interfaces, gate the machine:

- **Enabled** &ndash; when false, *charging allowed* is false, so the charger will not energize the vehicle (it stays in or returns to the paused sub-states) but the session and the pilot communication continue.
- **Available** &ndash; when set false the machine goes to state **F** and holds the pilot at &minus;12&nbsp;V. The vehicle sees the charger as switched off. Clearing it returns the machine to state **A**.

Both are covered in more detail in [Charging control](charging-control.md).

## Errors

When any error bit is set the reported state is **E**: the pilot is driven to &minus;12&nbsp;V and the contactor is opened.

| Error bit | Raised when | Recovery |
| --------- | ----------- | -------- |
| `PILOT_FAULT` | CP level invalid for the current state | Auto-clears after 60&nbsp;s |
| `DIODE_SHORT` | PWM active but the CP negative half does not reach &minus;12&nbsp;V | Auto-clears after 60&nbsp;s |
| `LOCK_FAULT` | Socket lock failed to reach the locked position | Latches |
| `UNLOCK_FAULT` | Socket lock failed to reach the unlocked position | Latches |
| `RCM_TRIGGERED` | Residual current monitor tripped | Auto-clears after 60&nbsp;s |
| `RCM_SELFTEST_FAULT` | RCM self-test (run on session start) failed | Auto-clears after 60&nbsp;s |
| `TEMPERATURE_HIGH` | Highest measured temperature above the threshold | Clears when temperature drops |
| `TEMPERATURE_FAULT` | Temperature sensor read error | Clears when the sensor reads again |

**Auto-clearing** errors start a **60&nbsp;second** timer; when it expires the bits are cleared and the machine resumes from state **A**. This gives transient faults (a noisy pilot reading, a momentary residual-current event) time to settle before retrying.

**Latching** errors (lock/unlock faults) are not cleared automatically because repeatedly driving a jammed lock actuator is undesirable. While a lock fault is active the firmware will not actuate the lock again, including the release on entering **E**, so the mechanism is left untouched until the situation is addressed and the device is manually restarted.

## Indicators

The optional charging and error LEDs follow the state directly:

| State        | Charging LED            | Error LED        |
| ------------ | ----------------------- | ---------------- |
| A            | off                     | off              |
| B1, B2       | slow flash (500/500&nbsp;ms) | off         |
| C1, D1       | brief blink (1900/100&nbsp;ms) | off       |
| C2, D2       | on                      | off              |
| E            | off                     | on               |
| F            | off                     | slow flash (500/500&nbsp;ms) |

## See also

- [Charging control](charging-control.md) &ndash; current selection, limits, authorization, enable/available.
- [Architecture](architecture.md) &ndash; how the state machine fits into the firmware.
- [Control Pilot](../10-hardware/control-pilot.md) and [Control Pilot calibration](../10-hardware/CP-calibration.md).
