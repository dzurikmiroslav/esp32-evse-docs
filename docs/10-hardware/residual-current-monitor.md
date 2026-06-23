---
title: Residual current monitor
---

## Overview

A **residual current monitor** (RCM) detects earth-fault currents &ndash; current that leaks to protective earth instead of returning through the neutral. In an EVSE this matters because AC and, in fault conditions, **DC** residual currents can occur, and ordinary Type&nbsp;A residual current devices do not reliably trip on DC leakage.

By integrating an RCM that can sense DC residual current, a charger can provide the DC fault detection that would otherwise require an expensive **Type&nbsp;B** RCD upstream. This was one of the original design goals of the project: getting the protection an EV installation needs without the cost of a Type&nbsp;B RCCB. The RCM is optional and must be disabled in settings on boards that do not have one.

## How it works

The board uses an RCM module (for example RCM14-01 / RCM14-03) that exposes two signals: a **trigger output** that asserts when residual current exceeds the module's threshold, and a **test input** that forces an internal fault so the device can be verified.

- **Trip.** While the RCM is enabled, the firmware watches the trigger line. If it asserts, an `RCM_TRIGGERED` error is raised and the charger goes to state **E**, opening the contactor.
- **Self-test.** At the start of every charging session the firmware exercises the RCM through its test input. If the module does not respond as expected, an `RCM_SELFTEST_FAULT` error is raised. This confirms the protection is working *before* energizing the vehicle, rather than trusting it blindly.

Both RCM errors **auto-clear** after about 60&nbsp;seconds, so a transient residual-current event causes a pause and retry rather than a permanent lockout. A genuine, persistent fault simply re-trips.

## Configuration

The RCM is declared in `board.yaml`:

```yaml
rcm:
  gpio: 41        # residual-current trigger input
  testGpio: 26    # self-test output
```

`gpio` is the trigger line read by the firmware; `testGpio` drives the module's self-test. Both must be present for the RCM to be usable. With the hardware fitted, enable the RCM in settings; without it, leave it disabled.

!!! danger
    The RCM is a personal-safety device. It supplements, and does not replace, the protective devices required by your local wiring rules. Always verify the self-test and a real trip during commissioning, and follow the applicable installation standards for earth-fault and PEN protection.

## See also

- [State machine](../20-software/state-machine.md#errors) &ndash; how RCM faults map to the error state.
- [ESP32-S2 EVSE DIY Alpha](esp32s2-evse-d-a.md) &ndash; a board with an RCM input.
