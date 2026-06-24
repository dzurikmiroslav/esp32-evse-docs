---
title: Socket lock
---

## Overview

A **socket lock** mechanically retains the plug in the charger's socket for the duration of a charging session. It is used on chargers with a **socket outlet** (detachable cable): when a vehicle is plugged in the lock engages so the plug cannot be pulled out while current may be flowing, and it releases when the session ends. A charger with a permanently attached cable does not use a socket lock.

The lock is a small **motorized actuator** of the type built into Type&nbsp;2 sockets. The firmware drives the actuator in two directions to lock and unlock, and reads a feedback contact to confirm the mechanism actually reached the commanded position.

## How it works

### Drive

The actuator motor can be driven through an H-bridge by **two control lines, A and B**. Their relative polarity sets the direction:

- **Lock** &ndash; drive A high and B low for the *operating time*, then release both lines (the motor coasts and the mechanism holds).
- **Unlock** &ndash; drive A low and B high for the *operating time*, then release both lines.

Driving the lines for a fixed time, rather than until a hard stop, is what limits the motor current; the actuator is designed to reach its end position within that window.

### Detection

To check the current position the firmware drives **both** control lines to the same level. With no voltage difference across it the motor does not move, while a position contact (a microswitch inside the actuator) can be read on a separate **detection input**. After a configurable *detection delay* the input is read and compared against the expected level for "locked".

Because actuators differ in which way their contact reads, a **detection polarity** setting selects whether the locked position corresponds to a logic high or a logic low on the detection input.

### Sequence and timing

A lock or unlock operation runs as its own task so it never blocks the [main control loop](../20-software/architecture.md#the-main-loop):

1. On a **lock** command the firmware waits a short **settle delay** (about 500&nbsp;ms) before the first attempt, giving the plug time to seat fully.
2. It drives the actuator for the **operating time**.
3. After the **detection delay** it reads the feedback contact.
4. If the position is not yet reached it **retries**, up to the configured *retry count*.
5. A minimum **break time** is enforced between consecutive operations so the motor is not driven again too soon.

If, after all retries, the actuator has not reached the locked position the status becomes a **locking fault**; the equivalent for release is an **unlocking fault**.

### Interaction with charging

The lock is operated by the [state machine](../20-software/state-machine.md):

- it **engages** when a session begins (entering state B1, i.e. when the vehicle is plugged in),
- it **releases** when the machine returns to state A (vehicle unplugged) or goes to E / F.

Charging will not start until the lock has settled to its idle (successfully locked) state &ndash; the *charging allowed* check requires it. A **lock fault** or **unlock fault** raises the corresponding error and puts the charger in state **E**. These faults latch: rather than repeatedly driving a jammed mechanism, the firmware stops actuating the lock until the problem is addressed and the device is restarted.

## Settings

The timing and detection behavior are adjustable at runtime (web UI, [Modbus](../20-software/Modbus.md), REST, scripts):

| Setting | Default | Notes |
| ------- | ------- | ----- |
| Operating time | 300&nbsp;ms | How long the motor is driven (range 100&ndash;1000&nbsp;ms) |
| Break time | 1000&nbsp;ms | Minimum rest between operations (not below the board's minimum) |
| Retry count | 5 | Attempts before declaring a fault |
| Detection polarity | &ndash; | Whether the locked position reads high or low |

## Configuration

The lock hardware is declared in `board.yaml`:

```yaml
socketLock:
  gpios: [20, 19]      # control lines A and B (H-bridge)
  detectionGpio: 34    # feedback contact input
  detectionDelay: 1000 # delay before reading the contact, in ms
  minBreakTime: 1000   # lower bound for the break time, in ms
```

`gpios` are the two motor control lines (A then B). `detectionGpio` is the position feedback input. `detectionDelay` is how long to wait after commanding a move before reading the contact, and `minBreakTime` is the floor for the user-configurable break time.

For how these terminals are wired on a real board, see the socket lock section of the [ESP32-S2 EVSE DIY Alpha](esp32s2-evse-d-a.md#socket-lock) as an example.

If you're building an EVSE without a socket-outlet (having the cable directly attached) omit this configuration from `board.yaml`.

!!! warning
    The socket lock is a safety feature: it prevents the plug being removed under load. Verify both locking and unlocking, and the fault behavior, before putting a socket-outlet charger into service. Make sure the actuator releases on power loss or fault so a vehicle can never be trapped.

!!! warning "CAUTION"
    Working with high voltage is dangerous. Always follow local laws and regulations regarding high voltage work. If you are unsure about the rules in your country, consult a licensed electrician for more information.

## See also

- [ESP32-S2 EVSE DIY Alpha](esp32s2-evse-d-a.md#socket-lock) &ndash; wiring and terminals.
- [Proximity Pilot](proximity-pilot.md) &ndash; the other half of socket-outlet operation.
- [State machine](../20-software/state-machine.md) &ndash; when the lock engages and releases.
