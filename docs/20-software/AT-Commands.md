---
title: AT Commands reference
---

AT commands are useful to communicate with the device from third party peripherals. A good use case is implememting a [HMI using ESPHome and LVGL](https://dzurikmiroslav.github.io/esp32-evse-docs/10-hardware/hmi-examples/#esphome-with-lvgl) which not only offers a nice touch screen interface, but also integrates natively the EVSE in Home Assistant.

AT commands support can be enabled in serial settings, any serial interface can be used for this. 

!!! note
    Only one interface can work in AT commands mode.

## Basic commands

When AT Commands module is enabled, at startup, the device will print automatically on the line the `RDY` message once, to inform the client that communication can start. Since this mostly happens when the board booted up, it can be useful to detect on the client side a freshly booted state.

### AT

Check that the communication is working properly.

| Command | Return Value | Return Code |
| ------- | ------------ | ----------- |
| `AT`    | -            | `OK`        |

### ATE

Echo back to the client the AT command input (see the entered commands on a serial console terminal).

| Command     | Return Value | Return Code |
| ----------- | ------------ | ----------- |
| `ATE<echo>` | -            | `OK`        |

| Parameter | Access | Type  | Description            |
| --------- | ------ | ----- | ---------------------- |
| `<echo>`  | WO     | uint8 | Echo commands (0 or 1) |

### AT+CMD

List all the supported AT commands by the ESP32-EVSE device.

| Command  | Return Value       | Return Code |
| -------- | ------------------ | ----------- |
| `AT+CMD` | *List of commands* | `OK`        |

Example:

```
AT+CMD

ATE0

ATE1

AT+RST

AT+CHIP?
AT+CHIP=?

AT+HEAP?
AT+HEAP=?

...

OK
```

### AT+SUB

Subscribe to periodic response to other commands, let ESP32-EVSE send responses automatically, repeatedly.

| Command                     | Return Value          | Return Code     |
| --------------------------- | --------------------- | --------------- |
| `AT+SUB=<command>,<period>` | -                     | `OK` \| `ERROR` |
| `AT+SUB=?`                  | *Command description* | `OK`            |

| Parameter   | Access | Type   | Description                     |
| ----------- | ------ | ------ | ------------------------------- |
| `<command>` | W      | string | AT command name (max length 32) |
| `<period>`  | W      | uint32 | Time period in ms               |

Example:

```
AT+SUB="+ENABLE",1000
OK
AT+SUB="+AVAILABLE",2000
OK
+ENABLE=1
+ENABLE=1
+AVAILABLE=1
+ENABLE=1
+ENABLE=1
+AVAILABLE=1
```

> **Note** 
> Only works with read commands (not write, test or execute).
> When you subscribe multiple times to the same command, only the period will be updated.
> AT commands task run in 100ms delay loop, setting any period of lower value has no effect.

### AT+UNSUB

Unsubscribe periodical reading of a certain command.

| Command              | Return Value          | Return Code |
| -------------------- | --------------------- | ----------- |
| `AT+UNSUB=<command>` | -                     | `OK`        |
| `AT+UNSUB=?`         | *Command description* | `OK`        |

| Parameter   | Access | Type   | Description                     |
| ----------- | ------ | ------ | ------------------------------- |
| `<command>` | W      | string | AT command name (max length 32) |

Example:

```
AT+SUB="+ENABLE",1000
OK
AT+SUB="+AVAILABLE",1000
OK
+ENABLE=1
+AVAILABLE=1
+ENABLE=1
+AVAILABLE=1
AT+UNSUB="+ENABLE"
+AVAILABLE=1
+AVAILABLE=1

AT+SUB="+ENABLE",1000
OK
AT+SUB="+AVAILABLE",1000
OK
+ENABLE=1
+AVAILABLE=1
AT+UNSUB=""
```

> **Note** 
> Call with empty string remove all command subscriptions.

## System commands

### AT+RST

Restart ESP32-EVSE device.

| Command  | Return Value | Return Code |
| -------- | ------------ | ----------- |
| `AT+RST` | -            | `OK`        |

### AT+UPTIME

Print time since last boot, in seconds.

| Command       | Return Value          | Return Code |
| ------------- | --------------------- | ----------- |
| `AT+UPTIME?`  | `+UPTIME=<uptime>`    | `OK`        |
| `AT+UPTIME=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<time>`  | RW     | uint32 | Time in s   |

### AT+CHIP

Print ESP32 chip model information.

| Command     | Return Value                       | Return Code |
| ----------- | ---------------------------------- | ----------- |
| `AT+CHIP?`  | `+CHIP=<model>,<cores>,<revision>` | `OK`        |
| `AT+CHIP=?` | *Command description*              | `OK`        |

| Parameter    | Access | Type   | Description                                                                                   |
| ------------ | ------ | ------ | --------------------------------------------------------------------------------------------- |
| `<model>`    | RO     | string | Name of model                                                                                 |
| `<cores>`    | RO     | uint8  | Number of CPU cores                                                                           |
| `<revision>` | RO     | uint16 | Chip revision number (in format MXX; where M - wafer major version, XX - wafer minor version) |

Example:
```
AT+CHIP?
+CHIP="esp32",2,100
OK
```

### AT+HEAP

Print memory heap status.

| Command     | Return Value           | Return Code |
| ----------- | ---------------------- | ----------- |
| `AT+HEAP?`  | `+HEAP=<used>,<total>` | `OK`        |
| `AT+HEAP=?` | *Command description*  | `OK`        |

| Parameter | Access | Type   | Description              |
| --------- | ------ | ------ | ------------------------ |
| `<used>`  | RO     | uint32 | Used heap size in bytes  |
| `<total>` | RO     | uint32 | Total heap size in bytes |

Example:
```
AT+HEAP?
+HEAP=115992,308884
OK
```

### AT+VER

Print ESP32-EVSE firmware version.

| Command    | Return Value          | Return Code |
| ---------- | --------------------- | ----------- |
| `AT+VER?`  | `+VER=<version>`      | `OK`        |
| `AT+VER=?` | *Command description* | `OK`        |

| Parameter   | Access | Type   | Description                                    |
| ----------- | ------ | ------ | ---------------------------------------------- |
| `<version>` | RO     | string | Version, commonly in format major.minor.bugfix |

Example:
```
AT+VER?
+VER="v2.0.0"
OK
```

### AT+IDFVER

Print ESP-IDF version that was used to build the firmware.

| Command       | Return Value          | Return Code |
| ------------- | --------------------- | ----------- |
| `AT+IDFVER?`  | `+IDFVER=<version>`   | `OK`        |
| `AT+IDFVER=?` | *Command description* | `OK`        |

| Parameter   | Access | Type   | Description     |
| ----------- | ------ | ------ | --------------- |
| `<version>` | RO     | string | ESP-IDF version |

Example:
```
AT+IDFVER?
+IDFVER="v5.5"
OK
```

### AT+BUILDTIME

Print the datetime of build.

| Command          | Return Value               | Return Code |
| ---------------- | -------------------------- | ----------- |
| `AT+BUILDTIME?`  | `+BUILDTIME=<date>,<time>` | `OK`        |
| `AT+BUILDTIME=?` | *Command description*      | `OK`        |

| Parameter | Access | Type   | Description  |
| --------- | ------ | ------ | ------------ |
| `<date>`  | RO     | string | Compile date |
| `<time>`  | RO     | string | Compile time |

Example:
```
AT+BUILDTIME?
+BUILDTIME="Aug 20 2025","22:13:07"
OK
```

### AT+TEMP

Print the measured temperature values.

| Command     | Return Value                 | Return Code |
| ----------- | ---------------------------- | ----------- |
| `AT+TEMP?`  | `+TEMP=<count>,<high>,<low>` | `OK`        |
| `AT+TEMP=?` | *Command description*        | `OK`        |

| Parameter | Access | Type  | Description                     |
| --------- | ------ | ----- | ------------------------------- |
| `<count>` | RO     | uint8 | Count of temperature sensors    |
| `<high>`  | RO     | int32 | Highest temperature in dg.C*100 |
| `<low>`   | RO     | int32 | Lowest temperature in dg.C*100  |

Example:
```
AT+TEMP?
+TEMP=2,2965,2887
OK

AT+TEMP?
+TEMP=1,2875,2875
OK

AT+TEMP?
+TEMP=0,0,0
OK
```
> **Note** 
> In the example above: first reading has 2 temperature sensors; second reading has 1 temperature sensor; last reading has no temperature sensor.

### AT+TZ

Read/write timezone name.

| Command            | Return Value          | Return Code     |
| ------------------ | --------------------- | --------------- |
| `AT+TZ?`           | `+TZ=<timezone>`      | `OK`            |
| `AT+TZ=<timezone>` | -                     | `OK` \| `ERROR` |
| `AT+TZ=?`          | *Command description* | `OK`            |

| Parameter    | Access | Type   | Description                   |
| ------------ | ------ | ------ | ----------------------------- |
| `<timezone>` | RW     | string | Timezone name (max length 32) |

Example:
```
AT+TZ?
+TZ="Etc/UTC"
OK

AT+TZ="Europe/Prague"
OK
```

### AT+TZRULE

Read Posix timezone string.

| Command       | Return Value              | Return Code |
| ------------- | ------------------------- | ----------- |
| `AT+TZRULE?`  | `+TZRULE=<timezone_rule>` | `OK`        |
| `AT+TZRULE=?` | *Command description*     | `OK`        |

| Parameter         | Access | Type   | Description                           |
| ----------------- | ------ | ------ | ------------------------------------- |
| `<timezone_rule>` | RO     | string | Posix timezone string (max length 32) |

Example:
```
AT+TZRULE?
+TZRULE="CET-1CEST,M3.5.0,M10.5.0/3"
OK
```

### AT+TIME

Read/write epoch Unix timestamp (write to set the system time manually).

| Command          | Return Value          | Return Code |
| ---------------- | --------------------- | ----------- |
| `AT+TIME?`       | `+TIME=<time>`        | `OK`        |
| `AT+TIME=<time>` | -                     | `OK`        |
| `AT+TIME=?`      | *Command description* | `OK`        |

| Parameter | Access | Type   | Description     |
| --------- | ------ | ------ | --------------- |
| `<time>`  | RW     | uint64 | Epoch time in s |

Example:
```
AT+TIME?
+TIME=1756129834
OK
```

## EVSE commands

### AT+STATE

Print ESP32-EVSE internal state according to J1772 standard.

| Command      | Return Value          | Return Code |
| ------------ | --------------------- | ----------- |
| `AT+STATE?`  | `+STATE=<state>`      | `OK`        |
| `AT+STATE=?` | *Command description* | `OK`        |

| Parameter | Access | Type  | Description                                                           |
| --------- | ------ | ----- | --------------------------------------------------------------------- |
| `<state>` | RO     | uint8 | EVSE state number (A=0, B1=1, B2=2, C1=3, C2=4, D1=5, D2=6, E=7, F=8) |

Example:
```
AT+STATE?
+STATE=4
OK
```

### AT+ERROR

Print ESP32-EVSE [error bits](https://github.com/dzurikmiroslav/esp32-evse/blob/2fbb62fade1f56bda89c5875871fef60d9c13a72/components/evse/include/evse.h#L11) in decimal value (use bitmask to decode).

| Command      | Return Value          | Return Code |
| ------------ | --------------------- | ----------- |
| `AT+ERROR?`  | `+ERROR=<error>`      | `OK`        |
| `AT+ERROR=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description    |
| --------- | ------ | ------ | -------------- |
| `<error>` | RO     | uint32 | EVSE error bit |

Example:
```
AT+ERROR?
+ERROR=0
OK
```

### AT+ENABLE

Read/write ESP32-EVSE charging enabled setting.

| Command              | Return Value          | Return Code |
| -------------------- | --------------------- | ----------- |
| `AT+ENABLE?`         | `+ENABLE=<enable>`    | `OK`        |
| `AT+ENABLE=<enable>` | -                     | `OK`        |
| `AT+ENABLE=?`        | *Command description* | `OK`        |

| Parameter  | Access | Type  | Description                 |
| ---------- | ------ | ----- | --------------------------- |
| `<enable>` | RW     | uint8 | Is enabled charger (0 or 1) |

Example:
```
AT+ENABLE?
+ENABLE=0
OK

AT+ENABLE=1
OK
```

### AT+AVAILABLE

Read/write ESP32-EVSE charging available setting (F state).

| Command                    | Return Value             | Return Code |
| -------------------------- | ------------------------ | ----------- |
| `AT+AVAILABLE?`            | `+AVAILABLE=<available>` | `OK`        |
| `AT+AVAILABLE=<available>` | -                        | `OK`        |
| `AT+AVAILABLE=?`           | *Command description*    | `OK`        |

| Parameter     | Access | Type  | Description                   |
| ------------- | ------ | ----- | ----------------------------- |
| `<available>` | RW     | uint8 | Is available charger (0 or 1) |

Example:
```
AT+AVAILABLE?
+AVAILABLE=0
OK
```

> **Note** 
> When ESP32-EVSE is set available=0 it not mean that state always will be F. Error can override the state to E.

### AT+REQAUTH

Read/write ESP32-EVSE require authorization option before starting to charge.

| Command                     | Return Value              | Return Code |
| --------------------------- | ------------------------- | ----------- |
| `AT+REQAUTH?`               | `+REQAUTH=<require_auth>` | `OK`        |
| `AT+REQAUTH=<require_auth>` | -                         | `OK`        |
| `AT+REQAUTH=?`              | *Command description*     | `OK`        |

| Parameter        | Access | Type  | Description                        |
| ---------------- | ------ | ----- | ---------------------------------- |
| `<require_auth>` | RW     | uint8 | Is required authorization (0 or 1) |

Example:
```
AT+REQAUTH?
+REQAUTH=0
OK
```

### AT+PENDAUTH

Print the state of the ESP32-EVSE pending authorization bit.


| Command         | Return Value               | Return Code |
| --------------- | -------------------------- | ----------- |
| `AT+PENDAUTH?`  | `+PENDAUTH=<pending_auth>` | `OK`        |
| `AT+PENDAUTH=?` | *Command description*      | `OK`        |

| Parameter        | Access | Type  | Description                       |
| ---------------- | ------ | ----- | --------------------------------- |
| `<pending_auth>` | RO     | uint8 | Is pending authorization (0 or 1) |

Example:
```
AT+PENDAUTH?
+PENDAUTH=1
OK
```

### AT+AUTH

This command is to confirm the pending authorization, and allow the system to charge.


| Command   | Return Value | Return Code |
| --------- | ------------ | ----------- |
| `AT+AUTH` | ---          | `OK`        |

Example:
```
AT+AUTH
OK
```

### AT+CHCUR

Read/write the actual charging current value. Value must be in range 6A - MAXCHCUR.

| Command              | Return Value          | Return Code     |
| -------------------- | --------------------- | --------------- |
| `AT+CHCUR?`          | `+CHCUR=<current>`    | `OK`            |
| `AT+CHCUR=<current>` | -                     | `OK` \| `ERROR` |
| `AT+CHCUR=?`         | *Command description* | `OK`            |

| Parameter   | Access | Type   | Description              |
| ----------- | ------ | ------ | ------------------------ |
| `<current>` | RW     | uint16 | Charging current in A*10 |

Example:
```
AT+CHCUR?
+CHCUR=60
OK

AT+CHCUR=20
ERROR
```
> **Note** 
> The example above returns error because value 2A is out of range (6A is lower limit)

### AT+DEFCHCUR

Read/write default charging current. Value must be in range 6A - MAXCHCUR.

| Command                 | Return Value          | Return Code     |
| ----------------------- | --------------------- | --------------- |
| `AT+DEFCHCUR?`          | `+DEFCHCUR=<current>` | `OK`            |
| `AT+DEFCHCUR=<current>` | -                     | `OK` \| `ERROR` |
| `AT+DEFCHCUR=?`         | *Command description* | `OK`            |

| Parameter   | Access | Type   | Description              |
| ----------- | ------ | ------ | ------------------------ |
| `<current>` | RW     | uint16 | Charging current in A*10 |

Example:
```
AT+DEFCHCUR?
+DEFCHCUR=320
OK
```

### AT+MAXCHCUR

Read/write max charging current. Value must be in range 6A - 63A.

| Command                 | Return Value          | Return Code     |
| ----------------------- | --------------------- | --------------- |
| `AT+MAXCHCUR?`          | `+MAXCHCUR=<current>` | `OK`            |
| `AT+MAXCHCUR=<current>` | -                     | `OK` \| `ERROR` |
| `AT+MAXCHCUR=?`         | *Command description* | `OK`            |

| Parameter   | Access | Type  | Description           |
| ----------- | ------ | ----- | --------------------- |
| `<current>` | RW     | uint8 | Charging current in A |

Example:
```
AT+MAXCHCUR?
+MAXCHCUR=63
OK
```

> **Note** 
> Set this to match the electrical limits of your installation!

### AT+CONSUMLIM

Read/write energy consumption limit (stop charging session if the limit has been hit).

| Command                      | Return Value               | Return Code |
| ---------------------------- | -------------------------- | ----------- |
| `AT+CONSUMLIM?`              | `+CONSUMLIM=<consumption>` | `OK`        |
| `AT+CONSUMLIM=<consumption>` | -                          | `OK`        |
| `AT+CONSUMLIM=?`             | *Command description*      | `OK`        |

| Parameter       | Access | Type   | Description       |
| --------------- | ------ | ------ | ----------------- |
| `<consumption>` | RW     | uint32 | Consumption in Wh |

Example:
```
AT+CONSUMLIM?
+CONSUMLIM=0
OK
```
> **Note** 
> Value 0 mean no limit

### AT+DEFCONSUMLIM

Read/write the default consumption limit.

| Command                         | Return Value                  | Return Code |
| ------------------------------- | ----------------------------- | ----------- |
| `AT+DEFCONSUMLIM?`              | `+DEFCONSUMLIM=<consumption>` | `OK`        |
| `AT+DEFCONSUMLIM=<consumption>` | -                             | `OK`        |
| `AT+DEFCONSUMLIM=?`             | *Command description*         | `OK`        |

| Parameter       | Access | Type   | Description       |
| --------------- | ------ | ------ | ----------------- |
| `<consumption>` | RW     | uint32 | Consumption in Wh |

Example:
```
AT+DEFCONSUMLIM?
+DEFCONSUMLIM=25000
OK
```
> **Note** 
> This example sets a 25kWh amount of energy consumption limit

> **Note** 
> Value 0 mean no limit

### AT+CHTIMELIM

Read/write charging time limit (stop charging session if the limit has been hit).

| Command               | Return Value          | Return Code |
| --------------------- | --------------------- | ----------- |
| `AT+CHTIMELIM?`       | `+CHTIMELIM=<time>`   | `OK`        |
| `AT+CHTIMELIM=<time>` | -                     | `OK`        |
| `AT+CHTIMELIM=?`      | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<time>`  | RW     | uint32 | Time in s   |

Example:
```
AT+CHTIMELIM=7200
OK
```

> **Note** 
> This example sets a 2h charging time limit

> **Note** 
> Value 0 mean no limit

### AT+DEFCHTIMELIM

Read/write the default charging time limit.

| Command                  | Return Value           | Return Code |
| ------------------------ | ---------------------- | ----------- |
| `AT+DEFCHTIMELIM?`       | `+DEFCHTIMELIM=<time>` | `OK`        |
| `AT+DEFCHTIMELIM=<time>` | -                      | `OK`        |
| `AT+DEFCHTIMELIM=?`      | *Command description*  | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<time>`  | RW     | uint32 | Time in s   |

Example:
```
AT+DEFCHTIMELIM?
+DEFCHTIMELIM=0
OK
```

> **Note** 
> Value 0 mean no limit

### AT+UNDERPOWERLIM

Read/write under power charging limit (stop charging session when power falls under the limit).

| Command                    | Return Value             | Return Code |
| -------------------------- | ------------------------ | ----------- |
| `AT+UNDERPOWERLIM?`        | `+UNDERPOWERLIM=<power>` | `OK`        |
| `AT+UNDERPOWERLIM=<power>` | -                        | `OK`        |
| `AT+UNDERPOWERLIM=?`       | *Command description*    | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<power>` | RW     | uint16 | Power in W  |

Example:
```
AT+UNDERPOWERLIM?
+UNDERPOWERLIM=0
OK
```

> **Note** 
> Value 0 mean no limit

### AT+DEFUNDERPOWERLIM

Read/write the default under power charging limit.

| Command                       | Return Value                | Return Code |
| ----------------------------- | --------------------------- | ----------- |
| `AT+DEFUNDERPOWERLIM?`        | `+DEFUNDERPOWERLIM=<power>` | `OK`        |
| `AT+DEFUNDERPOWERLIM=<power>` | -                           | `OK`        |
| `AT+DEFUNDERPOWERLIM=?`       | *Command description*       | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<power>` | RW     | uint16 | Power in W  |

Example:
```
AT+DEFUNDERPOWERLIM?
+DEFUNDERPOWERLIM=0
OK
```

> **Note** 
> Value 0 mean no limit

### AT+LIMREACH

Print the state of the charging limit bit (any limit was reached). 

| Command         | Return Value          | Return Code |
| --------------- | --------------------- | ----------- |
| `AT+LIMREACH?`  | `+LIMREACH=<reached>` | `OK`        |
| `AT+LIMREACH=?` | *Command description* | `OK`        |

| Parameter   | Access | Type  | Description                        |
| ----------- | ------ | ----- | ---------------------------------- |
| `<reached>` | R      | uint8 | Is charging limit reached (0 or 1) |

Example:
```
AT+LIMREACH?
+LIMREACH=0
OK
```

### AT+SOCKETOUTLET

Read/write socket outlet installation setting. Set it only for PP board. 
This enables detection of provided cable max current, and socket lock functionality (if it's available).

| Command                           | Return Value                    | Return Code     |
| --------------------------------- | ------------------------------- | --------------- |
| `AT+SOCKETOUTLET?`                | `+SOCKETOUTLET=<socket_outlet>` | `OK`            |
| `AT+SOCKETOUTLET=<socket_outlet>` | -                               | `OK` \| `ERROR` |
| `AT+SOCKETOUTLET=?`               | *Command description*           | `OK`            |

| Parameter         | Access | Type  | Description            |
| ----------------- | ------ | ----- | ---------------------- |
| `<socket_outlet>` | RW     | uint8 | Socket outlet (0 or 1) |

Example:
```
AT+SOCKETOUTLET?
+SOCKETOUTLET=0
OK
```

> **Note** 
> Value 0 mean no limit

## Energy meter commands

### AT+EMETERMODE

Read/write energy meter mode. 
Value CUR require `EMETER=(2|1),x`.
Value CUR_VLT require `EMETER=2,x`.

| Command                | Return Value          | Return Code     |
| ---------------------- | --------------------- | --------------- |
| `AT+EMETERMODE?`       | `+EMETERMODE=<mode>`  | `OK`            |
| `AT+EMETERMODE=<mode>` | -                     | `OK` \| `ERROR` |
| `AT+EMETERMODE=?`      | *Command description* | `OK`            |

| Parameter | Access | Type  | Description                             |
| --------- | ------ | ----- | --------------------------------------- |
| `<mode>`  | RW     | uint8 | Mode number (DUMMY=0, CUR=1, CUR_VLT=2) |

Example:
```
AT+EMETERMODE?
+EMETERMODE=1
OK

AT+EMETERMODE=5
ERROR
```

> **Note** 
> In the example above, value 5 fails because it is out of range

### AT+EMETERACVOLTAGE

Read/write energy meter voltage setting, used for calculation in DUMMY and CUR mode (use range 100V - 300V).

| Command                        | Return Value                 | Return Code     |
| ------------------------------ | ---------------------------- | --------------- |
| `AT+EMETERACVOLTAGE?`          | `+EMETERACVOLTAGE=<voltage>` | `OK`            |
| `AT+EMETERACVOLTAGE=<voltage>` | -                            | `OK` \| `ERROR` |
| `AT+EMETERACVOLTAGE=?`         | *Command description*        | `OK`            |

| Parameter   | Access | Type   | Description  |
| ----------- | ------ | ------ | ------------ |
| `<voltage>` | RW     | uint16 | Voltage in V |

Example:
```
AT+EMETERACVOLTAGE?
+EMETERACVOLTAGE=253
OK

AT+EMETERACVOLTAGE=305
ERROR
```

> **Note** 
> In the example above, setting value 305 fails because it is out of range

### AT+EMETERTHREEPHASE

Read/write energy meter three phase mode setting.

| Command                              | Return Value                       | Return Code |
| ------------------------------------ | ---------------------------------- | ----------- |
| `AT+EMETERTHREEPHASE?`               | `+EMETERTHREEPHASE=<three_phases>` | `OK`        |
| `AT+EMETERTHREEPHASE=<three_phases>` | -                                  | `OK`        |
| `AT+EMETERTHREEPHASE=?`              | *Command description*              | `OK`        |

| Parameter        | Access | Type  | Description           |
| ---------------- | ------ | ----- | --------------------- |
| `<three_phases>` | RW     | uint8 | Three phases (0 or 1) |

Example:
```
AT+EMETERTHREEPHASE?
+EMETERTHREEPHASE=1
OK
```

### AT+EMETERPOWER

Print the value of the actual charging power.

| Command            | Return Value           | Return Code |
| ------------------ | ---------------------- | ----------- |
| `AT+EMETERPOWER?`  | `+EMETERPOWER=<power>` | `OK`        |
| `AT+EMETERPOWER=?` | *Command description*  | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<power>` | RO     | uint16 | Power in W  |

Example:
```
AT+EMETERPOWER?
+EMETERPOWER=11085
OK
```

### AT+EMETERSESTIME

Print the duration of the current charging session.

| Command              | Return Value            | Return Code |
| -------------------- | ----------------------- | ----------- |
| `AT+EMETERSESTIME?`  | `+EMETERSESTIME=<time>` | `OK`        |
| `AT+EMETERSESTIME=?` | *Command description*   | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<time>`  | RO     | uint32 | Time in s   |

Example:
```
AT+EMETERSESTIME?
+EMETERSESTIME=3769
OK
```

### AT+EMETERCHTIME

Print the amount of time spent with charging.

| Command             | Return Value           | Return Code |
| ------------------- | ---------------------- | ----------- |
| `AT+EMETERCHTIME?`  | `+EMETERCHTIME=<time>` | `OK`        |
| `AT+EMETERCHTIME=?` | *Command description*  | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<time>`  | RO     | uint32 | Time in s   |

Example:
```
AT+EMETERCHTIME?
+EMETERCHTIME=3702
OK
```

### AT+EMETERCONSUM

Print the energy consumption of the actual session.

| Command             | Return Value                  | Return Code |
| ------------------- | ----------------------------- | ----------- |
| `AT+EMETERCONSUM?`  | `+EMETERCONSUM=<consumption>` | `OK`        |
| `AT+EMETERCONSUM=?` | *Command description*         | `OK`        |

| Parameter       | Access | Type   | Description       |
| --------------- | ------ | ------ | ----------------- |
| `<consumption>` | RO     | uint32 | Consumption in Wh |

Example:
```
AT+EMETERCONSUM?
+EMETERCONSUM=35547
OK
```

### AT+EMETERTOTCONSUM

Read/reset the total energy delivered with this ESP32-EVSE device.

| Command                            | Return Value                     | Return Code     |
| ---------------------------------- | -------------------------------- | --------------- |
| `AT+EMETERTOTCONSUM?`              | `+EMETERTOTCONSUM=<consumption>` | `OK`            |
| `AT+EMETERTOTCONSUM=<consumption>` | -                                | `OK` \| `ERROR` |
| `AT+EMETERTOTCONSUM=?`             | *Command description*            | `OK`            |

| Parameter       | Access | Type   | Description       |
| --------------- | ------ | ------ | ----------------- |
| `<consumption>` | RW     | uint64 | Consumption in Wh |

Example:
```
AT+EMETERTOTCONSUM?
+EMETERTOTCONSUM=915517547
OK

AT+EMETERCONSUM=0
OK

AT+EMETERCONSUM=5
ERROR
```
> **Note** 
> To reset total consumption use value 0, other values will throw error.


### AT+EMETERVOLTAGE

Print the measured or preset voltage.

| Command              | Return Value                                | Return Code |
| -------------------- | ------------------------------------------- | ----------- |
| `AT+EMETERVOLTAGE?`  | `+EMETERVOLTAGE=<l1_vlt>,<l2_vlt>,<l3_vlt>` | `OK`        |
| `AT+EMETERVOLTAGE=?` | *Command description*                       | `OK`        |

| Parameter  | Access | Type   | Description      |
| ---------- | ------ | ------ | ---------------- |
| `<l1_vlt>` | RO     | uint32 | L1 voltage in mV |
| `<l2_vlt>` | RO     | uint32 | L2 voltage in mV |
| `<l3_vlt>` | RO     | uint32 | L3 voltage in mV |

Example:
```
AT+EMETERVOLTAGE?
+EMETERVOLTAGE=255951,254753,255852
OK

AT+EMETERVOLTAGE?
+EMETERVOLTAGE=250000,0,0
OK
```

> **Note** 
> When energy meter is single phase, L2 and L3 voltage are 0.

### AT+EMETERCURRENT

Print the measured or preset current.

| Command              | Return Value                                | Return Code |
| -------------------- | ------------------------------------------- | ----------- |
| `AT+EMETERCURRENT?`  | `+EMETERCURRENT=<l1_cur>,<l2_cur>,<l3_cur>` | `OK`        |
| `AT+EMETERCURRENT=?` | *Command description*                       | `OK`        |

| Parameter  | Access | Type   | Description      |
| ---------- | ------ | ------ | ---------------- |
| `<l1_cur>` | RO     | uint32 | L1 current in mA |
| `<l2_cur>` | RO     | uint32 | L2 current in mA |
| `<l3_cur>` | RO     | uint32 | L3 current in mA |

Example:
```
AT+EMETERCURRENT?
+EMETERCURRENT=10951,10753,105852
OK

AT+EMETERCURRENT?
+EMETERCURRENT=32000,0,0
OK
```

> **Note** 
> When energy meter is single phase, L2 and L3 current are 0.

## Network commands

### AT+WIFISTACFG

This command is used to read/write WiFi STA configuration.

| Command                                     | Return Value                              | Return Code     |
| ------------------------------------------- | ----------------------------------------- | --------------- |
| `AT+WIFISTACFG?`                            | `+WIFISTACFG=<enabled>,<ssid>,<password>` | `OK`            |
| `AT+WIFISTACFG=<enabled>,<ssid>,<password>` | -                                         | `OK` \| `ERROR` |
| `AT+WIFISTACFG=?`                           | *Command description*                     | `OK`            |

| Parameter    | Access | Type   | Description                  |
| ------------ | ------ | ------ | ---------------------------- |
| `<enabled>`  | RW     | uint8  | Enabled/disable STA (0 or 1) |
| `<ssid>`     | RW     | string | SSID (max length 32)         |
| `<password>` | WO     | string | Password (max length 32)     |

Example:
```
AT+WIFISTACFG?
+WIFISTACFG=1,"MirkoTik-E12345",""
OK

AT+WIFISTACFG=0
OK

AT+WIFISTACFG?
+WIFISTACFG=0,"MirkoTik-E12345",""
OK

AT+WIFISTACFG=1,"MirkoTik-E99999","topsecret"
OK

AT+WIFISTACFG?
+WIFISTACFG=1,"MirkoTik-E99999",""
OK
```

> **Note** 
> Calling WIFISTACFG=(0|1) only disable/enable STA, SSID and password will be not modified.

### AT+WIFIAPCFG

Read/write WiFi access point mode configuration.

| Command                  | Return Value                  | Return Code |
| ------------------------ | ----------------------------- | ----------- |
| `AT+WIFIAPCFG?`          | `+WIFIAPCFG=<enabled>,<ssid>` | `OK`        |
| `AT+WIFIAPCFG=<enabled>` | -                             | `OK`        |
| `AT+WIFIAPCFG=?`         | *Command description*         | `OK`        |

| Parameter   | Access | Type   | Description                 |
| ----------- | ------ | ------ | --------------------------- |
| `<enabled>` | RW     | uint8  | Enabled/disable AP (0 or 1) |
| `<ssid>`    | RO     | string | SSID (max length 32)        |

Example:
```
AT+WIFIAPCFG?
+WIFIAPCFG=0,"ESP32-EVSE-03fc60"
OK

AT+WIFIAPCFG=1
OK

AT+WIFISTACFG?
+WIFIAPCFG=1,"ESP32-EVSE-03fb59"
OK
```

> **Note** 
> After enabling access point mode, if there's no client connected within 1 minute, access point mode will be disabled.

### AT+WIFISTACONN

Print WiFi client mode connection state.

| Command            | Return Value                       | Return Code |
| ------------------ | ---------------------------------- | ----------- |
| `AT+WIFISTACONN?`  | `+WIFISTACONN=<connection>,<rssi>` | `OK`        |
| `AT+WIFISTACONN=?` | *Command description*              | `OK`        |

| Parameter      | Access | Type  | Description             |
| -------------- | ------ | ----- | ----------------------- |
| `<connection>` | RO     | uint8 | Has connection (0 or 1) |
| `<rssi>`       | RO     | int8  | RSSI in dB              |

Example:
```
AT+WIFISTACONN?
+WIFISTACONN=1,-61
OK
```

### AT+WIFIAPCONN

Print WiFi access point mode connection state.

| Command           | Return Value               | Return Code |
| ----------------- | -------------------------- | ----------- |
| `AT+WIFIAPCONN?`  | `+WIFIAPCONN=<connection>` | `OK`        |
| `AT+WIFIAPCONN=?` | *Command description*      | `OK`        |

| Parameter      | Access | Type  | Description             |
| -------------- | ------ | ----- | ----------------------- |
| `<connection>` | RO     | uint8 | Has connection (0 or 1) |

Example:
```
AT+WIFIAPCONN?
+WIFIAPCONN=0
OK
```

### AT+WIFISTASTATIC

This command is used to read/write WiFi STA static IP configuration.

| Command                                                    | Return Value                                              | Return Code     |
| ---------------------------------------------------------- | --------------------------------------------------------- | --------------- |
| `AT+WIFISTASTATIC?`                                        | `+WIFISTASTATIC=<enabled>,<ip>,<gateway>,<netmask>,<dns>` | `OK`            |
| `AT+WIFISTASTATIC=<enabled>,<ip>,<gateway>,<netmask><dns>` | -                                                         | `OK` \| `ERROR` |
| `AT+WIFISTASTATIC=?`                                       | *Command description*                                     | `OK`            |

| Parameter   | Access | Type   | Description                        |
| ----------- | ------ | ------ | ---------------------------------- |
| `<enabled>` | RW     | uint8  | Enabled/disable static IP (0 or 1) |
| `<ip>`      | RW     | string | IP address (max length 32)         |
| `<gateway>` | RW     | string | Gateway (max length 32)            |
| `<netmask>` | RW     | string | Netmask (max length 32)            |
| `<dns>`     | RW     | string | DNS server (max length 32)         |

Example:
```
OK
AT+WIFISTASTATIC?
+WIFISTASTATIC=1,"192.168.88.100","192.168.88.1","255.255.255.0","192.168.88.1"
OK

AT+WIFISTASTATIC=0
OK

AT+WIFISTASTATIC?
+WIFISTASTATIC=0,"192.168.88.100","192.168.88.1","255.255.255.0","192.168.88.1"
OK

AT+WIFISTASTATIC=1,"192.168.88.101"
OK
```

> **Note** 
> Calling WIFISTASTATIC=(0|1) only disable/enable static IP, other parameters will be not modified. 

### AT+WIFISTAIP

Print WiFi client IP address.

| Command          | Return Value          | Return Code |
| ---------------- | --------------------- | ----------- |
| `AT+WIFISTAIP?`  | `+WIFISTAIP=<ip>`     | `OK`        |
| `AT+WIFISTAIP=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<ip>`    | RO     | string | IP address  |

Example:
```
AT+WIFISTAIP?
+WIFISTAIP="192.168.88.50"
OK
```

### AT+WIFISTAMAC

Print WiFi client MAC address.

| Command           | Return Value          | Return Code |
| ----------------- | --------------------- | ----------- |
| `AT+WIFISTAMAC?`  | `+WIFISTAMAC=<mac>`   | `OK`        |
| `AT+WIFISTAMAC=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<mac>`   | RO     | string | IP address  |

Example:
```
AT+WIFISTAMAC?
+WIFISTAMAC="30:c6:f7:x3:ab:5c"
OK
```

### AT+WIFIAPIP

Print WiFi access point IP address.

| Command         | Return Value          | Return Code |
| --------------- | --------------------- | ----------- |
| `AT+WIFIAPIP?`  | `+WIFIAPIP=<ip>`      | `OK`        |
| `AT+WIFIAPIP=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<ip>`    | RO     | string | IP address  |

Example:
```
AT+WIFIAPIP?
+WIFIAPIP="192.168.4.1"
OK
```

### AT+WIFIAPMAC

Print WiFi access point MAC address.

| Command          | Return Value          | Return Code |
| ---------------- | --------------------- | ----------- |
| `AT+WIFIAPMAC?`  | `+WIFIAPMAC=<mac>`    | `OK`        |
| `AT+WIFIAPMAC=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description |
| --------- | ------ | ------ | ----------- |
| `<mac>`   | RO     | string | IP address  |

Example:
```
AT+WIFIAPMAC?
+WIFIAPMAC="30:c6:f7:x3:ab:5d"
OK
```

### AT+WIFIAPSCAN

Scan WiFi for surrounding acc esspoints.

| Command         | Return Value                       | Return Code |
| --------------- | ---------------------------------- | ----------- |
| `AT+WIFIAPSCAN` | `+WIFIAPSCAN=<ssid>,<auth>,<rssi>` | `OK`        |

| Parameter | Access | Type   | Description                        |
| --------- | ------ | ------ | ---------------------------------- |
| `<ssid>`  | RO     | string | AP SSID (max length 32)            |
| `<auth>`  | RO     | uint8  | AP require authentication (0 or 1) |
| `<rssi>`  | RO     | int8   | AP RSSI in dB                      |

Example:
```
AT+WIFIAPSCAN
+WIFIAPSCAN="MirkoTik-E12345",1,-51
+WIFIAPSCAN="Pretty fly for a WiFi",1,-69
+WIFIAPSCAN="Winternet is Coming",1,-72
+WIFIAPSCAN="The LAN Before Time",1,-75
OK
```

### AT+HOSTNAME

Read/write mDNS hostname.

| Command                  | Return Value           | Return Code     |
| ------------------------ | ---------------------- | --------------- |
| `AT+HOSTNAME?`           | `+HOSTNAME=<hostname>` | `OK`            |
| `AT+HOSTNAME=<hostname>` | -                      | `OK` \| `ERROR` |
| `AT+HOSTNAME=?`          | *Command description*  | `OK`            |

| Parameter    | Access | Type   | Description              |
| ------------ | ------ | ------ | ------------------------ |
| `<hostname>` | RO     | string | Hostname (max length 32) |

Example:
```
AT+HOSTNAME?
+HOSTNAME="evse"
OK

AT+HOSTNAME="evse2"
OK
```

### AT+INSTNAME

Read/write mDNS instance name.

| Command                       | Return Value                | Return Code     |
| ----------------------------- | --------------------------- | --------------- |
| `AT+INSTNAME?`                | `+INSTNAME=<instance_name>` | `OK`            |
| `AT+INSTNAME=<instance_name>` | -                           | `OK` \| `ERROR` |
| `AT+INSTNAME=?`               | *Command description*       | `OK`            |

| Parameter         | Access | Type   | Description                   |
| ----------------- | ------ | ------ | ----------------------------- |
| `<instance_name>` | RO     | string | Instance name (max length 32) |

Example:
```
AT+INSTNAME?
+INSTNAME="evse"
OK

AT+INSTNAME="evse2"
OK
```

## Serial commands

### AT+SERIAL

Read/write serial configuration.

| Command                                                             | Return Value                                                      | Return Code |
| ------------------------------------------------------------------- | ----------------------------------------------------------------- | ----------- |
| `AT+SERIAL<id>?`                                                    | `+SERIAL<id>=<mode>,<baud_rate>,<data_bits>,<stop_bits>,<parity>` | `OK`        |
| `AT+SERIAL<id>=<mode>,<baud_rate>,<data_bits>,<stop_bits>,<parity>` | -                                                                 | `OK`        |
| `AT+SERIAL<id>=?`                                                   | *Command description*                                             | `OK`        |

| Parameter     | Access | Type   | Description                                                      |
| ------------- | ------ | ------ | ---------------------------------------------------------------- |
| `<id>`        | RW     | uint8  | Serial id (0,1,2?, some chips has only 2 uart)                   |
| `<mode>`      | RW     | uint8  | Serial mode (NONE=0, LOG=1, MODBUS=2, NEXTION=3, SCRIPT=4, AT=5) |
| `<baud_rate>` | RW     | uint32 | Serial baud rate                                                 |
| `<data_bits>` | RW     | uint8  | Serial data bits (enum values uart_word_length_t)                |
| `<stop_bits>` | RW     | uint8  | Serial stop bits (enum values uart_stop_bits_t)                  |
| `<parity>`    | RW     | uint8  | Serial parity (enum values uart_parity_t)                        |

Example:
```
AT+SERIAL0?
+SERIAL0=0,115200,3,1,0
OK
```

> **Note** 
> Remember which serial port you are using for AT commands. After changing the currently used serial port, you will not be able to revert the changes from here.

## Board config

The commands below provide functionality to read board config.

### AT+DEVNAME

Print device name.

| Command        | Return Value             | Return Code |
| -------------- | ------------------------ | ----------- |
| `AT+DEVNAME?`  | `+DEVNAME=<device_name>` | `OK`        |
| `AT+DEVNAME=?` | *Command description*    | `OK`        |

| Parameter       | Access | Type   | Description                 |
| --------------- | ------ | ------ | --------------------------- |
| `<device_name>` | RO     | string | Device name (max length 32) |

Example:
```
AT+DEVNAME?
+DEVNAME="ESP32-DevKitC EVSE"
OK
```

### AT+SOCKETLOCK

Print the definition of socket lock.

| Command           | Return Value                | Return Code |
| ----------------- | --------------------------- | ----------- |
| `AT+SOCKETLOCK?`  | `+SOCKETLOCK=<socket_lock>` | `OK`        |
| `AT+SOCKETLOCK=?` | *Command description*       | `OK`        |

| Parameter       | Access | Type  | Description                      |
| --------------- | ------ | ----- | -------------------------------- |
| `<socket_lock>` | RO     | uint8 | Has defined socket lock (0 or 1) |

Example:
```
AT+SOCKETLOCK?
+SOCKETLOCK=1
OK
```

### AT+SOCKETLOCKMINBREAKTIME

Print socket lock minimum break time.

| Command                       | Return Value                               | Return Code |
| ----------------------------- | ------------------------------------------ | ----------- |
| `AT+SOCKETLOCKMINBREAKTIME?`  | `+SOCKETLOCKMINBREAKTIME=<min_break_time>` | `OK`        |
| `AT+SOCKETLOCKMINBREAKTIME=?` | *Command description*                      | `OK`        |

| Parameter          | Access | Type   | Description                      |
| ------------------ | ------ | ------ | -------------------------------- |
| `<min_break_time>` | RO     | uint16 | Socket lock min break time in ms |

Example:
```
AT+SOCKETLOCKMINBREAKTIME?
+SOCKETLOCKMINBREAKTIME=1
OK
```

### AT+PROXIMITY

Print the definition of a proximity pilot.

| Command          | Return Value             | Return Code |
| ---------------- | ------------------------ | ----------- |
| `AT+PROXIMITY?`  | `+PROXIMITY=<proximity>` | `OK`        |
| `AT+PROXIMITY=?` | *Command description*    | `OK`        |

| Parameter     | Access | Type  | Description                          |
| ------------- | ------ | ----- | ------------------------------------ |
| `<proximity>` | RO     | uint8 | Has defined proximity pilot (0 or 1) |

Example:
```
AT+PROXIMITY?
+PROXIMITY=1
OK
```

### AT+TEMPSENSOR

Print the definition of a temperature sensor bus.

| Command           | Return Value                       | Return Code |
| ----------------- | ---------------------------------- | ----------- |
| `AT+TEMPSENSOR?`  | `+TEMPSENSOR=<temperature_sensor>` | `OK`        |
| `AT+TEMPSENSOR=?` | *Command description*              | `OK`        |

| Parameter              | Access | Type  | Description                                 |
| ---------------------- | ------ | ----- | ------------------------------------------- |
| `<temperature_sensor>` | RO     | uint8 | Has defined temperature sensor bus (0 or 1) |

Example:
```
AT+TEMPSENSOR?
+TEMPSENSOR=1
OK
```

> **Note** 
> When you have defined temperature sensor bus, it doesn't automaticaly mean you have a connected temperature sensor.
> Command AT+TEMP returns the count of the active temperature sensors.

### AT+EMETER

Print the energy meter settings.

| Command       | Return Value                    | Return Code |
| ------------- | ------------------------------- | ----------- |
| `AT+EMETER?`  | `+EMETER=<type>,<three_phases>` | `OK`        |
| `AT+EMETER=?` | *Command description*           | `OK`        |

| Parameter        | Access | Type  | Description                                                 |
| ---------------- | ------ | ----- | ----------------------------------------------------------- |
| `<type>`         | RO     | uint8 | Type of energy meter (NONE=0, CURRENT=1, CURRENT_VOLTAGE=2) |
| `<three_phases>` | RO     | uint8 | Three phases energy meter (0 or 1)                          |

Example:
```
AT+TEMPSENSOR?
+EMETER=2,0
OK
```

### AT+SERIALS

Print the serial definitions.

| Command        | Return Value             | Return Code |
| -------------- | ------------------------ | ----------- |
| `AT+SERIALS?`  | `+SERIALS=<type>,<name>` | `OK`        |
| `AT+SERIALS=?` | *Command description*    | `OK`        |

| Parameter | Access | Type   | Description                      |
| --------- | ------ | ------ | -------------------------------- |
| `<type>`  | RO     | uint8  | Type of serial (UART=1, RS485=2) |
| `<name>`  | RO     | string | Name of serial (max length 32)   |

Example:
```
AT+SERIALS?
+SERIALS=1,"UART via USB"
+SERIALS=2,"RS485"
+SERIALS=1,"UART"
OK
```

### AT+AUXINPUTS

Print AUX input definitions.

| Command          | Return Value          | Return Code |
| ---------------- | --------------------- | ----------- |
| `AT+AUXINPUTS?`  | `+AUXINPUTS=<name>`   | `OK`        |
| `AT+AUXINPUTS=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description                  |
| --------- | ------ | ------ | ---------------------------- |
| `<name>`  | RO     | string | Name of input (max length 8) |

Example:
```
AT+AUXINPUTS?
+AUXINPUTS="IN1"
+AUXINPUTS="IN2"
OK
```

### AT+AUXOUTPUTS

Print AUX output definitions.

| Command           | Return Value          | Return Code |
| ----------------- | --------------------- | ----------- |
| `AT+AUXOUTPUTS?`  | `+AUXOUTPUTS=<name>`  | `OK`        |
| `AT+AUXOUTPUTS=?` | *Command description* | `OK`        |

| Parameter | Access | Type   | Description                   |
| --------- | ------ | ------ | ----------------------------- |
| `<name>`  | RO     | string | Name of output (max length 8) |

Example:
```
AT+AUXOUTPUTS?
+AUXOUTPUTS="OUT1"
+AUXOUTPUTS="OUT2"
+AUXOUTPUTS="OUT3"
+AUXOUTPUTS="OUT4"
OK
```

### AT+AUXANALOGINPUTS

Print AUX analog input definitions.

| Command                | Return Value              | Return Code |
| ---------------------- | ------------------------- | ----------- |
| `AT+AUXANALOGINPUTS?`  | `+AUXANALOGINPUTS=<name>` | `OK`        |
| `AT+AUXANALOGINPUTS=?` | *Command description*     | `OK`        |

| Parameter | Access | Type   | Description                         |
| --------- | ------ | ------ | ----------------------------------- |
| `<name>`  | RO     | string | Name of analog input (max length 8) |

Example:
```
AT+AUXANALOGINPUTS?
+AUXANALOGINPUTS="IN3"
OK
```