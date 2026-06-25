---
title: REST API / WebDAV
---

# REST API

The firmware exposes a JSON REST API served by the same HTTP server as the web
interface, on **port 80**. It is the same API the bundled web UI uses, so
anything the UI can do is available over REST.

All endpoints live under the base path:

```
http://<device-ip>/api/v1
```

## Authentication

Authentication is **HTTP Basic** and is **optional**. When no username and no
password are set (the default), every request is accepted. As soon as a
non-empty credential pair is stored, all endpoints require it.

```
Authorization: Basic <base64(user:password)>
```

A failed or missing credential returns `401 Unauthorized` with a
`WWW-Authenticate: Basic realm="Users"` header.

Credentials are stored in NVS. If you lock yourself out, hold the device button
to factory-reset (this erases all NVS configuration, not only the credentials).

See [Set credentials](#credentials) to change them over the API.

## Conventions

| Aspect | Behaviour |
| ------ | --------- |
| Read endpoints | `GET`, respond with `application/json` (a few return `text/plain`, noted below) |
| Write endpoints | `POST`; on success respond `200` with the plain-text body `OK` |
| Request bodies | Must be JSON with header `Content-Type: application/json` |
| Max request body | 50 KB (larger bodies are rejected with `413`) |
| Partial config writes | Object endpoints apply only the keys present; at least one known key must be supplied |

### Status codes

| Code | Meaning |
| ---- | ------- |
| `200` | Success |
| `400` | Bad request — malformed JSON, wrong value type, out-of-range value, or no recognised field |
| `401` | Authentication required / failed |
| `404` | Unknown endpoint, or the requested resource does not exist (e.g. firmware check with nothing available) |
| `413` | Request body larger than 50 KB |
| `415` | Body sent without `Content-Type: application/json` |
| `500` | Internal error |

A handful of long-running operations return non-standard codes in the `5xx`
range to signal a specific failure (firmware upgrade `521`, firmware receive
`522`, invalid firmware image `523`, Nextion upload `531`/`532`, OTA version
lookup `520`, memory allocation `512`).

## Quick start

```bash
# read full charging state
curl http://192.168.4.1/api/v1/state

# start charging
curl -X POST http://192.168.4.1/api/v1/state/enabled \
     -H 'Content-Type: application/json' -d 'true'

# set charging current to 10 A
curl -X POST http://192.168.4.1/api/v1/state/charging-current \
     -H 'Content-Type: application/json' -d '10'

# with authentication
curl -u admin:secret http://192.168.4.1/api/v1/info
```

## Endpoint overview

### State & charging control

| Method | Path | Body | Purpose |
| ------ | ---- | ---- | ------- |
| `GET` | `/state` | — | Full live charging state |
| `POST` | `/state` | object | Set several state fields at once |
| `POST` | `/state/enabled` | boolean | Enable / disable charging |
| `POST` | `/state/available` | boolean | Put the EVSE in / out of service |
| `POST` | `/state/charging-current` | number (A) | Set the active charging current |
| `POST` | `/state/consumption-limit` | number (Ws) | Session energy limit |
| `POST` | `/state/charging-time-limit` | number (s) | Session time limit |
| `POST` | `/state/under-power-limit` | number (W) | Auto-stop under-power threshold |
| `POST` | `/state/authorize` | — | Authorize a pending charge |
| `POST` | `/state/reset-total-consumption` | — | Reset the lifetime energy counter |

### Configuration

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `GET`/`POST` | `/config/evse` | Charging hardware & defaults |
| `GET`/`POST` | `/config/wifi` | Wi-Fi station settings |
| `GET`/`POST` | `/config/discovery` | mDNS hostname / instance name |
| `GET`/`POST` | `/config/serial` | Serial port modes |
| `GET`/`POST` | `/config/modbus` | Modbus TCP / unit id |
| `GET`/`POST` | `/config/script` | Lua scripting toggles |
| `GET`/`POST` | `/config/scheduler` | NTP, timezone, time schedules |
| `GET` | `/config` | All of the above in one object |

### Wi-Fi

| Method | Path | Body | Purpose |
| ------ | ---- | ---- | ------- |
| `GET` | `/wifi/scan` | — | Scan for access points |
| `GET` | `/wifi/state` | — | STA / AP addresses and signal |
| `POST` | `/wifi/state/ap` | boolean | Start / stop the access point |

### Scripting

| Method | Path | Body | Purpose |
| ------ | ---- | ---- | ------- |
| `GET` | `/script/output` | — | Script log output (text) |
| `POST` | `/script/reload` | — | Reload scripts |
| `GET` | `/script/components` | — | List script components |
| `GET` | `/script/components/{id}` | — | Component parameters |
| `POST` | `/script/components/{id}` | object | Set component parameters |

### Firmware & system

| Method | Path | Body | Purpose |
| ------ | ---- | ---- | ------- |
| `GET` | `/firmware/channels` | — | Available OTA channels |
| `GET`/`POST` | `/firmware/channel` | string | Current OTA channel |
| `GET` | `/firmware/check-update` | — | Compare current vs. available version |
| `POST` | `/firmware/update` | — | OTA update from the selected channel |
| `POST` | `/firmware/upload` | binary | Upload a firmware image |
| `GET` | `/nextion/info` | — | Connected Nextion HMI info |
| `POST` | `/nextion/upload` | binary | Upload a Nextion TFT file |
| `GET` | `/info` | — | Firmware / chip / heap info |
| `GET` | `/board-config` | — | Capabilities declared in `board.yaml` |
| `GET`/`POST` | `/time` | number | Read / set the system clock (Unix epoch) |
| `POST` | `/restart` | — | Restart the device |
| `POST` | `/credentials` | object | Set HTTP Basic credentials |
| `GET` | `/log` | — | Runtime log (text) |
| `GET`/`DELETE` | `/log/panic` | — | Read / clear the last panic log (text) |

## State

### `GET /state`

```json
{
  "state": "C2",
  "available": true,
  "enabled": true,
  "pendingAuth": false,
  "limitReached": false,
  "chargingCurrent": 16.0,
  "consumptionLimit": 0,
  "chargingTimeLimit": 0,
  "underPowerLimit": 0,
  "errors": null,
  "sessionTime": 1820,
  "chargingTime": 1740,
  "consumption": 9320000,
  "totalConsumption": 412900000,
  "power": 3680,
  "voltage": [231.4, 0.0, 0.0],
  "current": [16.0, 0.0, 0.0]
}
```

| Field | Type | Notes |
| ----- | ---- | ----- |
| `state` | string | IEC 61851 / J1772 state: `A`, `B1`, `B2`, `C1`, `C2`, `D1`, `D2`, `E`, `F`. See [State machine](state-machine.md) |
| `available` | boolean | `false` = taken out of service (state F) |
| `enabled` | boolean | Charging enabled |
| `pendingAuth` | boolean | Waiting for authorization before charging |
| `limitReached` | boolean | A session limit stopped the charge |
| `chargingCurrent` | number | Active current offered to the vehicle, in A (0.1 A resolution) |
| `consumptionLimit` | number | Session energy limit in Ws (`0` = none) |
| `chargingTimeLimit` | number | Session time limit in s (`0` = none) |
| `underPowerLimit` | number | Auto-stop power threshold in W (`0` = none) |
| `errors` | null or array | `null` when healthy, otherwise a list of error strings (see below) |
| `sessionTime` | number | Seconds since the session started (cable connected) |
| `chargingTime` | number | Seconds actually charging |
| `consumption` | number | Session energy in Ws |
| `totalConsumption` | number | Lifetime energy in Ws |
| `power` | number | Instantaneous power in W |
| `voltage` | number[3] | L1–L3 voltage in V |
| `current` | number[3] | L1–L3 current in A |

Possible `errors` values: `pilot_fault`, `diode_short`, `lock_fault`,
`unlock_fault`, `rcm_triggered`, `rcm_selftest_fault`, `temperature_high`,
`temperature_fault`.

### `POST /state`

Sets one or more live values at once. Any subset of the following is accepted;
at least one must be present.

```json
{
  "available": true,
  "enabled": true,
  "chargingCurrent": 10,
  "consumptionLimit": 0,
  "chargingTimeLimit": 0,
  "underPowerLimit": 0
}
```

### Single-value state endpoints

These take a **bare JSON scalar** as the body, not an object.

| Path | Body | Example |
| ---- | ---- | ------- |
| `POST /state/enabled` | boolean | `true` |
| `POST /state/available` | boolean | `false` |
| `POST /state/charging-current` | number (A) | `13.5` |
| `POST /state/consumption-limit` | number (Ws) | `10000000` |
| `POST /state/charging-time-limit` | number (s) | `7200` |
| `POST /state/under-power-limit` | number (W) | `200` |

`POST /state/authorize` and `POST /state/reset-total-consumption` take **no
body** — the first clears a pending authorization so charging can begin, the
second zeroes the lifetime energy counter.

## Configuration

### `GET`/`POST` `/config/evse`

```json
{
  "maxChargingCurrent": 16,
  "defaultChargingCurrent": 10.0,
  "requireAuth": false,
  "socketOutlet": false,
  "rcm": false,
  "temperatureThreshold": 60,
  "defaultConsumptionLimit": 0,
  "defaultChargingTimeLimit": 0,
  "defaultUnderPowerLimit": 0,
  "socketLockOperatingTime": 3000,
  "socketLockBreakTime": 1000,
  "socketLockDetectionHigh": false,
  "socketLockRetryCount": 3,
  "energyMeterMode": "cur",
  "energyMeterAcVoltage": 230,
  "energyMeterThreePhases": false
}
```

| Field | Type | Notes |
| ----- | ---- | ----- |
| `maxChargingCurrent` | integer | Hard ceiling in A, range **6–63** |
| `defaultChargingCurrent` | number | Current applied at boot, A (0.1 A resolution) |
| `requireAuth` | boolean | Require [authorization](#single-value-state-endpoints) before charging |
| `socketOutlet` | boolean | Socket-outlet mode (proximity sensing + socket lock) vs. fixed cable |
| `rcm` | boolean | Residual current monitor enabled |
| `temperatureThreshold` | number | Throttle/stop temperature in °C |
| `defaultConsumptionLimit` | integer | Default session energy limit in Ws |
| `defaultChargingTimeLimit` | integer | Default session time limit in s |
| `defaultUnderPowerLimit` | integer | Default under-power threshold in W |
| `socketLockOperatingTime` | integer | Motor drive time in ms |
| `socketLockBreakTime` | integer | Pause between lock attempts in ms |
| `socketLockDetectionHigh` | boolean | Sense polarity (`true` = locked reads high) |
| `socketLockRetryCount` | integer | Lock/unlock retry attempts |
| `energyMeterMode` | string | `dummy`, `cur` (current only), or `cur_vlt` (current + voltage) |
| `energyMeterAcVoltage` | integer | Assumed AC voltage in V when voltage is not measured |
| `energyMeterThreePhases` | boolean | Three-phase metering |

!!! note
    Writes are validated against the board's real capabilities. For example you
    cannot select `cur_vlt` on a board without voltage sensing, or a
    `maxChargingCurrent` outside 6–63 A — such writes return `400`.

### `GET`/`POST` `/config/wifi`

```json
{
  "enabled": true,
  "ssid": "myhome",
  "staticEnabled": false,
  "staticIp": "",
  "staticGateway": "",
  "staticNetmask": "",
  "staticDns": ""
}
```

For `POST`, add a `password` field. The `GET` response never returns the
password.

!!! warning
    The new Wi-Fi configuration is applied about one second **after** the
    response is sent, so the HTTP reply still reaches you over the connection
    that is about to change. If the new settings are wrong the device may become
    unreachable on the network.

### `GET /wifi/state`

```json
{
  "ip": "192.168.1.50",
  "mac": "AA:BB:CC:DD:EE:FF",
  "rssi": -57,
  "apIp": "192.168.4.1",
  "apMac": "AA:BB:CC:DD:EE:F0",
  "apEnabled": true,
  "apSsid": "esp32-evse"
}
```

### `GET /wifi/scan`

```json
[
  { "ssid": "myhome", "rssi": -57, "auth": true },
  { "ssid": "guest",  "rssi": -71, "auth": false }
]
```

### `POST /wifi/state/ap`

Body is a bare boolean: `true` starts the access point, `false` stops it
(applied after a short delay, as with the Wi-Fi config).

### `GET`/`POST` `/config/discovery`

```json
{ "hostname": "evse", "instanceName": "Garage EVSE" }
```

mDNS hostname (`<hostname>.local`) and the human-readable instance name.

### `GET`/`POST` `/config/serial`

An **array**, one entry per serial port that the board declares, in board
order. A `POST` must contain exactly the same number of entries as the `GET`
returns, otherwise it is rejected.

```json
[
  { "mode": "log",    "baudRate": 115200, "dataBits": "8", "stopBits": "1", "parity": "disable" },
  { "mode": "modbus", "baudRate": 9600,   "dataBits": "8", "stopBits": "1", "parity": "even" }
]
```

| Field | Values |
| ----- | ------ |
| `mode` | `none`, `at`, `log`, `nextion`, `modbus`, `script` |
| `baudRate` | integer |
| `dataBits` | `5`, `6`, `7`, `8` |
| `stopBits` | `1`, `1.5`, `2` |
| `parity` | `disable`, `even`, `odd` |

### `GET`/`POST` `/config/modbus`

```json
{ "tcpEnabled": false, "unitId": 1 }
```

`tcpEnabled` starts the Modbus TCP server on **port 502**; `unitId` is the
slave/unit id, range **1–255**. See [Modbus](Modbus.md).

### `GET`/`POST` `/config/script`

```json
{ "enabled": false, "autoReload": false }
```

Toggles the [Lua](Lua.md) engine and automatic reload on file change.

### `GET`/`POST` `/config/scheduler`

```json
{
  "ntpEnabled": true,
  "ntpServer": "pool.ntp.org",
  "ntpFromDhcp": false,
  "timezone": "CET-1CEST,M3.5.0,M10.5.0/3",
  "schedules": [
    {
      "action": "ch_cur_6a",
      "mon": 16777215, "tue": 16777215, "wed": 16777215,
      "thu": 16777215, "fri": 16777215, "sat": 0, "sun": 0
    }
  ]
}
```

| Field | Type | Notes |
| ----- | ---- | ----- |
| `ntpEnabled` | boolean | Enable the NTP client |
| `ntpServer` | string | NTP server hostname |
| `ntpFromDhcp` | boolean | Take the NTP server from DHCP |
| `timezone` | string | POSIX TZ string; an unknown value returns `400` |
| `schedules` | array | List of scheduled actions |

Each schedule has an `action` and one numeric field per weekday (`mon`…`sun`).
Each weekday value is a **24-bit hour mask**: bit *n* set means the action
applies during hour *n* (bit 0 = 00:00–01:00 … bit 23 = 23:00–24:00). So
`16777215` (`0xFFFFFF`) means "all day".

Actions: `enable`, `available`, `ch_cur_6a`, `ch_cur_8a`, `ch_cur_10a`.

### `GET /config`

Returns every configuration block in one object, keyed by `evse`, `wifi`,
`discovery`, `serial`, `modbus`, `script`, `scheduler`. There is no combined
`POST` — write each block to its own endpoint.

## Scripting

### `GET /script/output`

Plain-text script log. The response carries an `X-Count` header with the total
number of buffered lines; pass `?index=<n>` to fetch from a given line, for
incremental polling.

### `GET /script/components` / `GET /script/components/{id}`

```json
[
  { "id": "solar", "name": "Solar surplus", "description": "Follow PV export" }
]
```

```json
{
  "params": [
    { "key": "min_current", "name": "Minimum current", "type": "number", "value": 6 },
    { "key": "enabled",     "name": "Enabled",          "type": "boolean", "value": true }
  ]
}
```

A non-existent id returns JSON `null`. To set parameters, `POST` to
`/script/components/{id}` with a `params` array of `{ "key": ..., "value": ... }`
(value type may be string, number or boolean). `POST /script/reload` reloads all
scripts. See [Lua](Lua.md).

## Firmware & OTA

### `GET /firmware/channels` / `GET`/`POST` `/firmware/channel`

`channels` returns the channel names declared in `board.yaml`. `channel`
reads (string) or sets (bare JSON string body) the active channel.

### `GET /firmware/check-update`

```json
{ "available": "v1.2.0", "current": "v1.1.0" }
```

Returns `404` if the latest-version info cannot be fetched. Compare
`available` with `current` yourself to decide whether to update.

### `POST /firmware/update`

Triggers an HTTPS OTA download from the selected channel and, if the available
version differs from the running one, flashes it and restarts. No body. Failure
modes: `520` (cannot fetch version info), `521` (upgrade failed).

### `POST /firmware/upload`

Upload a firmware binary directly as the raw request body
(`application/octet-stream`). The image header is validated against the running
project name; a mismatched or too-short image returns `523`. On success the
device restarts into the new image.

```bash
curl -X POST --data-binary @esp32-evse.bin \
     http://192.168.4.1/api/v1/firmware/upload
```

## Nextion HMI

### `GET /nextion/info`

```json
{ "model": "NX4832T035", "touch": true, "flashSize": 16777216 }
```

Returns JSON `null` if no display answers. See [Nextion](Nextion.md).

### `POST /nextion/upload`

Upload a `.tft` file as the raw body. Optional query parameter
`?baud-rate=<n>` (default `921600`) sets the upload baud rate. Errors `531`
(write failed) / `532` (receive failed).

## System & info

### `GET /info`

```json
{
  "uptime": 86400,
  "appVersion": "v1.1.0",
  "appDate": "Feb 21 2025",
  "appTime": "07:13:00",
  "idfVersion": "v5.4.1",
  "chip": "esp32",
  "chipCores": 2,
  "chipRevision": 3,
  "heap": { "allocated": 120000, "free": 90000, "largestFreeBlock": 60000, "minFree": 70000 },
  "temperatureSensorCount": 1,
  "temperatureLow": 24.5,
  "temperatureHigh": 41.0
}
```

`uptime` is in seconds, temperatures in °C, heap figures in bytes.

### `GET /board-config`

Reports what the board's `board.yaml` actually provides — useful to decide
which features to expose in a client.

```json
{
  "deviceName": "ESP32-S2 EVSE",
  "socketLock": true,
  "proximity": true,
  "socketLockMinBreakTime": 1000,
  "rcm": true,
  "temperatureSensor": true,
  "energyMeter": "cur_vlt",
  "energyMeterThreePhases": true,
  "serials": [
    { "type": "uart",  "name": "UART" },
    { "type": "rs485", "name": "RS485" }
  ],
  "auxInputs": ["AUX_IN1"],
  "auxOutputs": ["AUX_OUT1"],
  "auxAnalogInputs": []
}
```

`energyMeter` is one of `none`, `cur`, `cur_vlt`; serial `type` is `none`,
`uart` or `rs485`. See [Board config schema](Board-config-schema.md).

### `GET`/`POST` `/time`

The body is a bare number — Unix epoch seconds. `GET` returns the current
clock; `POST` sets it and immediately re-evaluates the schedules.

```bash
curl -X POST http://192.168.4.1/api/v1/time \
     -H 'Content-Type: application/json' -d "$(date +%s)"
```

### `POST /restart`

Schedules a restart about one second later. No body.

### `POST /credentials`

```json
{ "user": "admin", "password": "secret" }
```

Sets the HTTP Basic credentials used by every endpoint. Sending **empty
strings** for both disables authentication.

### `GET /log` and `GET`/`DELETE /log/panic`

`GET /log` streams the runtime log as plain text, with an `X-Count` header and
optional `?index=<n>` for incremental reads (same scheme as
[`/script/output`](#get-scriptoutput)). `GET /log/panic` returns the stored
panic/crash log as plain text with an `X-Time` header (timestamp of the panic);
`DELETE /log/panic` clears it.

## WebDAV file access

Script files and other contents of the device filesystem are **not** served
through this REST API. They are exposed over **WebDAV** on the same server,
under `/dav/`. Point any WebDAV-capable file browser at:

```
dav://<device-ip>/dav/
```

It supports `PROPFIND`, `GET`, `PUT`, `DELETE`, `MKCOL`, `COPY` and `MOVE`, so
you can edit Lua scripts in place.

!!! warning
    Unlike the REST API and the web interface, the WebDAV endpoints are **not**
    authenticated. The HTTP Basic credentials set via
    [`/credentials`](#credentials) gate the REST API and web UI only; the WebDAV
    handlers perform no credential check. Anyone who can reach the device on the
    network can read, upload, overwrite or delete files under `/dav/`
    (including Lua scripts), even when credentials are configured. Keep the
    device on a trusted network.

## Examples

### EVCC Integration

See [Integration Examples](integration.md) how to interconnect with EVCC using the REST API.

## See also

- [Integration Examples](integration.md)
- [Modbus](Modbus.md)
- [Nextion](Nextion.md)
- [AT commands](AT-Commands.md)
- [Lua](Lua.md)
