---
title: Integration examples
---

## Home Assistant with autodiscovery

Use a [LUA Script](Lua.md) to publish the charger state, power, session energy, temperature, error states as MQTT topics,
and accept commands to enable/disable charging and to set the charging current. Optionally announce everything
to Home Assistant via MQTT discovery so the entities appear automatically.

Set the values in the script directly overwriting the defaults or from the web UI: Scripting -> components -> "MQTT bridge to Home Assistant". 

**EVSE identifier** is important in case of autodiscovery, as it is used as topic prefix, as node path to uniquely identify the charger and when generating entity ids.

![Home Assistant](/images/luascript_mqtt_homeassistant.png)

File `/usr/lua/ha_bridge.lua`:

```lua
local evse = require("evse")
local energymeter = require("energymeter")
local mqtt = require("mqtt")
local json = require("json")

local STATE_NAMES = {
    [evse.STATEA]  = "A",
    [evse.STATEB1] = "B1",
    [evse.STATEB2] = "B2",
    [evse.STATEC1] = "C1",
    [evse.STATEC2] = "C2",
    [evse.STATED1] = "D1",
    [evse.STATED2] = "D2",
    [evse.STATEE]  = "E",
    [evse.STATEF]  = "F",
}

local function truthy(data)
    data = tostring(data):lower()
    return data == "on" or data == "1" or data == "true"
end

local function sanitize_identifier(value)
    local s = tostring(value or ""):lower()
    if not s:match("^[a-z0-9]+$") then
        error("invalid EVSE identifier '" .. tostring(value) ..
              "': use only a-z and 0-9 (no spaces, dashes, dots or other characters)")
    end
    return s
end

return {
    id = "ha_bridge",
    name = "MQTT bridge to Home Assistant",
    description = "Home Assistant MQTT bridge with entity autodiscovery",
    params = {
        uri = { type = "string",  name = "Broker URI", default = "mqtt://192.168.1.10:1883" },
        username = { type = "string",  name = "Username", default = "user" },
        password = { type = "string",  name = "Password", default = "pass" },
        identifier = { type = "string",  name = "EVSE identifier", default = "evse" },
        interval = { type = "number",  name = "Publish interval (s)", default = 5 },
        discovery = { type = "boolean", name = "Publish HA discovery", default = true },
    },

    start = function(params)
        print("starting Home Assistant bridge...")

        local raw = params.identifier
        if raw == nil or raw == "" then raw = "evse" end
        local id = sanitize_identifier(raw)
        local interval_ms = math.max(1, math.floor(params.interval or 10)) * 1000

        local state_topic = id .. "/state"
        local cmd_topic = id .. "/+/set"

        -- empty credential fields -> nil (anonymous broker)
        local user = (params.username ~= "") and params.username or nil
        local pass = (params.password ~= "") and params.password or nil

        local client = mqtt.client(params.uri, user, pass)

        -- Home Assistant discovery: one device, all entities read the single
        -- state topic through value_template. Config topics are retained.
        local function announce()
            local device = { identifiers = { id }, name = id, manufacturer = "esp32-evse", model = "esp32-evse" }
            -- slug becomes the entity id suffix: <domain>.<id>_<slug>
            local function disco(kind, slug, cfg)
                cfg.object_id = id .. "_" .. slug -- HA derives entity_id from object_id
                cfg.unique_id = id .. "_" .. slug
                cfg.device = device
                client:publish("homeassistant/" .. kind .. "/" .. id .. "/" .. slug .. "/config",
                    json.encode(cfg), 0, 1)
            end

            disco("sensor", "state", {
                name = "State", state_topic = state_topic,
                value_template = "{{ value_json.state }}", icon = "mdi:ev-station" })
            disco("sensor", "power", {
                name = "Power", state_topic = state_topic,
                value_template = "{{ value_json.power }}",
                unit_of_measurement = "W", device_class = "power", state_class = "measurement" })
            disco("sensor", "session_energy", {
                name = "Session energy", state_topic = state_topic,
                value_template = "{{ value_json.session_energy }}",
                unit_of_measurement = "Wh", device_class = "energy", state_class = "total_increasing" })
            disco("sensor", "temperature", {
                name = "Temperature", state_topic = state_topic,
                value_template = "{{ value_json.temperature }}",
                unit_of_measurement = "°C", device_class = "temperature", state_class = "measurement" })
            disco("binary_sensor", "charging", {
                name = "Charging", state_topic = state_topic,
                value_template = "{{ 'ON' if value_json.charging else 'OFF' }}",
                payload_on = "ON", payload_off = "OFF", device_class = "battery_charging" })
            disco("binary_sensor", "error", {
                name = "Error", state_topic = state_topic,
                value_template = "{{ 'ON' if value_json.error else 'OFF' }}",
                payload_on = "ON", payload_off = "OFF", device_class = "problem" })
            disco("switch", "charging_enabled", {
                name = "Charging enabled", state_topic = state_topic,
                value_template = "{{ 'ON' if value_json.enabled else 'OFF' }}", icon = "mdi:ev-plug-type2",
                command_topic = id .. "/enabled/set",
                payload_on = "ON", payload_off = "OFF", state_on = "ON", state_off = "OFF" })
            disco("number", "charging_current_limit", {
                name = "Charging current limit", state_topic = state_topic,
                value_template = "{{ value_json.current }}", icon = "mdi:current-ac",
                command_topic = id .. "/current/set",
                min = 6, max = evse.getmaxchargingcurrent(), step = 1,
                unit_of_measurement = "A", mode = "slider" })
        end

        return coroutine.create(function()
            repeat
                print("connecting...")
            until client:connect()
            print("connected!")

            if params.discovery then announce() end

            -- one subscription with a wildcard; dispatch by the middle segment
            client:subscribe(cmd_topic, function(topic, data)
                -- topic is "<id>/<what>/set" -> extract <what>
                local what = string.sub(topic, #id + 2)
                what = string.sub(what, 1, #what - 4) -- strip "/set"

                local methods = {
                    enabled = function() evse.setenabled(truthy(data)) end,
                    current = function()
                        local amps = tonumber(data)
                        if amps then
                            -- setchargingcurrent raises on out-of-range values
                            if not pcall(evse.setchargingcurrent, amps) then
                                print("ha_bridge: rejected current '" .. tostring(data) .. "'")
                            end
                        end
                    end,
                }
                if methods[what] then methods[what]() end
            end)

            while true do
                local state = evse.getstate()
                local payload = {}
                payload.state = STATE_NAMES[state] or "?"
                payload.charging = state == evse.STATEC2 or state == evse.STATED2
                payload.enabled = evse.getenabled()
                payload.error = evse.geterror() > 0
                payload.power = energymeter.getpower()
                payload.current = evse.getchargingcurrent()
                payload.session_energy = energymeter.getconsumption()
                payload.temperature = evse.gethightemperature()

                client:publish(state_topic, json.encode(payload), 0, 1) -- retained
                coroutine.yield(interval_ms)
            end
        end)
    end
}
```

!!! note
    Don't forget to add `component.register(require("ha_bridge"))` to your `/usr/lua/init.lua` file.

## ThingsBoard cloud MQTT

Implementing an integration to ThingsBoard cloud with a [LUA Script](Lua.md):

File `/usr/lua/thingsboard.lua`:

```lua
local evse = require("evse")
local energymeter = require("energymeter")
local mqtt = require("mqtt")
local json = require("json")

return {
    id = "thingsboard",
    name = "ThingsBooard",
    description = "ThingsBooard MQTT connector",
    params = {
        token = { type = "string", name = "Access token" }
    },
    start = function(params)
        print("starting thingsboard connector...")

        if not params.token then
            error("require access token")
        end

        local client = mqtt.client("mqtt://demo.thingsboard.io:1883", params.token)

        return coroutine.create(function()
            repeat
                print("connecting...")
            until client:connect()
            print("conected!")            

            client:subscribe("v1/devices/me/rpc/request/+", function(topic, data)
                local payload = json.decode(data)
                local reqid = string.sub(topic, #"v1/devices/me/rpc/request/" + 1)
                local restopic = "v1/devices/me/rpc/response/" .. reqid

                function response(value)
                    client:publish(restopic, json.encode({ value = value }))
                end

                local methods = {
                    ["getEnabled"] = function()
                        response(evse.getenabled())
                    end,
                    ["setEnabled"] = function(param)
                        evse.setenabled(param)
                    end,
                    ["getChargingCurrent"] = function()
                        response(evse.getchargingcurrent())
                    end,
                    ["setChargingCurrent"] = function(param)
                        evse.setchargingcurrent(param)
                    end
                }

                if methods[payload.method] ~= nil then
                    methods[payload.method](payload.params)
                end
            end)

            while true do
                local payload = {}
                local state = evse.getstate()
                payload["session"] = state >= evse.STATEB1 and state <= evse.STATED2
                payload["charging"] = state == evse.STATEC2 or state == evse.STATED2
                payload["enabled"] = evse.getenabled()
                payload["available"] = evse.getavailable()
                payload["error"] = evse.geterror() > 0
                payload["consumption"] = energymeter.getconsumption()
                payload["power"] = energymeter.getpower()
                payload["temperature"] = evse.gethightemperature()
                payload["uptime"] = os.clock()
                payload["currentL1"], payload["currentL2"], payload["currentL3"] = energymeter.getcurrent()
                payload["voltageL1"], payload["voltageL3"], payload["voltageL3"] = energymeter.getvoltage()

                client:publish("v1/devices/me/telemetry", json.encode(payload))

                coroutine.yield(60000)
            end
        end)
    end
}
```

!!! note
    Don't forget to add `component.register(require("thingsboard"))` to your `/usr/lua/init.lua` file.

## EVCC

Here's a mapping between ESP32-EVSE REST Api and EVCC. Replace `EVSE_IP` with your charger's address:

```yaml
chargers:
  - name: esp32evse
    type: custom                      # continuous current control + built-in meter

    # status: map ESP32-EVSE state (A,B1,B2,C1,C2,D1,D2,E,F) to EVCC's A..F
    status:
      source: http
      uri: http://EVSE_IP/api/v1/state
      jq: .state[0:1]                 # first letter only: B1/B2->B, C1/C2->C, D1/D2->D

    enabled:                          # is charging enabled? -> bool
      source: http
      uri: http://EVSE_IP/api/v1/state
      jq: .enabled

    enable:                           # start/stop charging
      source: http
      uri: http://EVSE_IP/api/v1/state/enabled
      method: POST
      headers:
        - content-type: application/json   # required, else the firmware returns 415
      body: "{{if .enable}}true{{else}}false{{end}}"

    maxcurrent:                       # integer A
      source: http
      uri: http://EVSE_IP/api/v1/state/charging-current
      method: POST
      headers:
        - content-type: application/json
      body: "{{.maxcurrent}}"

    maxcurrentmillis:                 # fractional A -> "dimmable"; device rounds to 0.1 A
      source: http
      uri: http://EVSE_IP/api/v1/state/charging-current
      method: POST
      headers:
        - content-type: application/json
      body: "{{.maxcurrentmillis}}"

    # ---- built-in meter ----
    power:
      source: http
      uri: http://EVSE_IP/api/v1/state
      jq: .power                      # W
    energy:
      source: http
      uri: http://EVSE_IP/api/v1/state
      jq: .totalConsumption / 3600000 # Ws -> kWh (lifetime, monotonic)
    currents:                         # must be exactly three entries
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .current[0] }
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .current[1] }
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .current[2] }
    voltages:
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .voltage[0] }
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .voltage[1] }
      - { source: http, uri: "http://EVSE_IP/api/v1/state", jq: .voltage[2] }

loadpoints:
  - title: Garage
    charger: esp32evse
    mode: pv
    mincurrent: 6
    maxcurrent: 16                    # <= the device's maxChargingCurrent
    phases: 1                         # set to your wiring (1 or 3)
```

!!! note
    This hasn't been thoroughly tested. Please [give feedback](https://github.com/dzurikmiroslav/esp32-evse/discussions) if you try it.

!!! note    
    ESP32-EVSE stores charging current at 0.1 A (deci-amp) granularity, so the "milliamp" path really gives 100 mA steps.
    It's genuinely continuously dimmable, just not to 1 mA. 
    
    Energy uses `totalConsumption`, not `consumption`. EVCC treats the charger's energy as a monotonic counter and diffs
    it for session energy; `consumption` resets to 0 each session and would produce negative diffs, so use the lifetime counter.
    
    Set ESP32-EVSE so it doesn't require authorization for charging. If HTTP Basic credentials are set, add an auth: block
    (type: basic, user:, password:) to each plugin. 

## ESPHome

The [AT commands engine](AT-Commands.md) can be used to communicate from other open source projects like ESPHome with ESP32-EVSE. 

The [external ESPHome component](https://github.com/nagyrobi/esp32evse-esphome) implements accessing most EVSE parameters as ESPHome
sensors, binary sensors, text sensors, switches, buttons, number inputs. These can appear in ESPHome and Home Assistant as native components.

It is intended to be used on a separate ESP32 microcontroller connected to ESP32-EVSE thorugh an UART. The biggest advantage here is that
ESP32-EVSE doesn't have to deal with any of the above, can keep working independently as a mission critical piece of hardware, all this
functionality can run separately without generating extra load on the charger.

## See also

- [Modbus](Modbus.md)
- [Nextion](Nextion.md)
- [AT commands](AT-Commands.md)
- [Lua](Lua.md)
- [REST API](rest-api.md)