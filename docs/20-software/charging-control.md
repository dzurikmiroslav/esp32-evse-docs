---
title: Charging control
---

## Overview

This page covers the operational controls the [state machine](state-machine.md) consumes: how the charging current is chosen and advertised, the optional limits that end a session, and the gates &ndash; authorization, *enabled* and *available* &ndash; that decide whether the charger energizes the vehicle. All of these settings are stored in NVS and can be changed at runtime from the web UI, the [REST API](openapi.md), [Modbus](Modbus.md), [Lua scripts](Lua.md) or [AT commands](AT-Commands.md).

## Charging current

The charger advertises an available current to the vehicle by varying the duty cycle of the 1&nbsp;kHz Control Pilot PWM (see [Control Pilot](../10-hardware/control-pilot.md)). Several values interact to decide what is actually advertised:

- **Maximum charging current** &ndash; the hard ceiling for the installation, set to the rating of the protective breaker (or cable). Range **6&ndash;63&nbsp;A**. Stored in NVS.
- **Charging current** &ndash; the working set-point, in 0.1&nbsp;A steps, never below **6.0&nbsp;A** and never above the maximum. This is the value a load-management script or external controller adjusts on the fly.
- **Default charging current** &ndash; the value applied at start-up, stored in NVS.
- **Cable rating** &ndash; in socket-outlet mode the maximum current the attached cable supports is read from the [Proximity Pilot](../10-hardware/proximity-pilot.md) when the vehicle is plugged in.

The advertised current is the **charging current capped by the cable rating**. Lowering the maximum or the cable rating immediately lowers what is advertised; the vehicle reacts within a second or two by reducing its draw.

!!! warning
    The maximum charging current protects the wiring. Set it to the value of the circuit breaker protecting the EVSE, or the cable's maximum current if that is lower. The firmware will advertise up to this value; nothing downstream enforces it for you.

### Dynamic load management

There is no built-in grid-side load balancing, but the charging current can be steered continuously by writing the charging-current value from an external source. A [Lua script](Lua.md) can read another meter (over a serial input, an analog input, or MQTT) and lower the charging current &ndash; or pause charging &ndash; when the rest of the building load is high, then raise it again. Modbus and the REST API expose the same control for an external controller.

## Charging limits

Three optional limits end an in-progress session. When a limit is reached, the session moves from the charging sub-state to the paused one (for example `C2` to `C1`); the vehicle stays connected and charging resumes if the condition clears.

| Limit | Unit | Behavior |
| ----- | ---- | -------- |
| Consumption limit | Wh | Stop after this much energy has been delivered this session |
| Charging time limit | s | Stop after this session duration |
| Under-power limit | W | Stop when delivered power stays below this value for 60&nbsp;seconds continuously |

The **under-power** limit is how the charger detects that a vehicle has finished: a full battery tapers to a trickle, the power drops below the threshold, and after 60&nbsp;seconds of staying low the session is stopped. This is useful when some EVs start decreasing charging power when reaching 80-90% SoC, one may want to stop charging at this level to preserve battery life: charging the battery only up to a set energy so it stops around 80&nbsp;%.

Each limit has a *default* value stored in NVS (applied at start-up) and a *live* value for the current session. A value of zero disables that limit. A *limit reached* flag is exposed to the external interfaces.

## Gates: authorization, enabled, available

Three independent conditions decide whether the charger will actually energize the vehicle. They are evaluated together as part of the *charging allowed* check in the [state machine](state-machine.md#transitions).

### Authorization

When **require authorization** is on, a session does not start charging until it is explicitly authorized. Authorization is granted from any interface (web UI, REST, Modbus, Lua, AT) and is valid for **60&nbsp;seconds**; it must coincide with the vehicle being plugged in, and it is reset on every disconnect so each plug-in needs a fresh grant. This is the hook for access control &ndash; for example an RFID reader driven by a script that authorizes only known cards. While the charger is waiting, a *pending authorization* flag is exposed.

### Enabled

The **enabled** flag pauses or resumes energy delivery without disturbing the vehicle connection or the pilot dialogue. When disabled, the charger stays connected and keeps communicating but does not close the contactor (it remains in, or returns to, the paused sub-state). This is the natural control for tariff-based charging &ndash; enable during the low tariff, disable otherwise.

### Available

The **available** flag switches the whole charger on or off as seen by the vehicle. When set unavailable the state machine goes to state **F** and holds the pilot at &minus;12&nbsp;V, so the vehicle treats the charger as switched off. Clearing it returns the charger to normal operation.

The difference between *enabled* and *available* matters: a disabled charger still negotiates with the vehicle and can resume in seconds, while an unavailable charger presents itself as off.

This is useful to temporarily render the EVSE to be out of order of usage.

## See also

- [State machine](state-machine.md) &ndash; how these controls gate the charging sequence.
- [Control Pilot](../10-hardware/control-pilot.md) &ndash; how current is advertised.
- [Proximity Pilot](../10-hardware/proximity-pilot.md) &ndash; cable current rating in socket-outlet mode.
- [Energy metering](../10-hardware/energy-metering.md) &ndash; where consumption and power come from.
- [Modbus reference](Modbus.md) &ndash; register map for all of the above.
