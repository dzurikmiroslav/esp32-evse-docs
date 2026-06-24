---
title: ModBUS reference
---

Controller support ModBUS server (slave) over TCP/IP and RTU/serial.

In the web interface, ModBUS TCP server can be enabled/disabled and set a Unit ID.
ModBUS TCP is listening on port `502`.

![ModBUS settings](/images/web-settings-modbus.png "ModBUS settings")

ModBUS RTU can be set in serial settings, any serial interface (UART, RS-485) can be operating in ModBUS slave mode.

!!! note
    Only one interface can work in ModBUS RTU mode.

![Serial settings](/images/web-settings-serial.png "Serial settings")

## ModBUS registers

| Register Address | Number Of Registers | Access | Description | Representation |
|---|---|---|---|---|
| 100 | 1 | R | EVSE state (A, B1, B2, C1, C2, D1, D2, E, F) | char[2] |
| 101 | 2 | R | EVSE error bits | uint32 |
| 103 | 1 | R/W | Charging enabled (enabled=1, disabled=0) | uint16 |
| 104 | 1 | R/W | Charger available (available=1, available=0) | uint16 |
| 105 | 1 | R | Pending authorization before start of charging, when authorization is required (1 when pending otherwise 0) | uint16 |
| 106 | 1 | R/W | Charging current in A*10 | uint16 |
| 107 | 2 | R/W | Consumption limit in Wh | uint32 |
| 109 | 2 | R/W | Charging time limit in s | uint32 |
| 111 | 1 | R/W | Underpower limit in W | uint16 |
| 112 | 1 | W | Authorize to start charging when is pending (value 1 must be written) | uint16 |
| 200 | 1 | R | Charging power in W | uint16 |
| 201 | 2 | R | Session time in s | uint32 |
| 203 | 2 | R | Charging time in s | uint32 |
| 205 | 2 | R | Consumption in Wh | uint32 |
| 207 | 2 | R | L1 voltage in mV | uint32 |
| 209 | 2 | R | L2 voltage in mV | uint32 |
| 211 | 2 | R | L3 voltage in mV | uint32 |
| 213 | 2 | R | L1 current in mA | uint32 |
| 215 | 2 | R | L2 current in mA | uint32 |
| 217 | 2 | R | L3 current in mA | uint32 |
| 300 | 1 | R/W | Socket outlet (enabled=1, disabled=0) | uint16 |
| 301 | 1 | R/W | RCM (enabled=1, disabled=0) | uint16 |
| 302 | 1 | R/W | Temperature threshold in dg.C | uint16 |
| 303 | 1 | R/W | Require authorization to start charging (enabled=1, disabled=0) | uint16 |
| 304 | 1 | R/W | Max charging current in A, stored in NVS | uint16 |
| 305 | 1 | R/W | Default charging current in A*10, stored in NVS | uint16 |
| 306 | 2 | R/W | Default consumption limit in Wh, stored in NVS | uint32 |
| 308 | 2 | R/W | Default charging time limit in s, stored in NVS | uint32 |
| 310 | 1 | R/W | Default underpower limit in W, stored in NVS | uint16 |
| 311 | 1 | R/W | Socket lock operating time in ms | uint16 |
| 312 | 1 | R/W | Socket lock break time in ms | uint16 |
| 313 | 1 | R/W | Socket lock detection (unlock_high=0, locked_high=1) | uint16 |
| 314 | 1 | R/W | Socket lock retry count | uint16 |
| 315 | 1 | R/W | Energy meter mode (DUMMY=0, CUR=1, CUR_VLT=2) | uint16 |
| 316 | 1 | R/W | Energy meter voltage in V, when is not measured | uint16 |
| 317 | 1 | R/W | Energy meter three phases (enabled=1, disabled=0) | uint16 |
| 400 | 2 | R | Uptime in s | uint32
| 402 | 1 | R | Low temperature in dg.C*100 | int16 |
| 403 | 1 | R | High temperature in dg.C*100 | int16 |
| 404 | 1 | R | Temperature sensor count | uint16 |
| 405 | 16 | R | App version | char[16] |
| 421 | 1 | W | Restart (value 1 must be written) | uint16 |

!!! note
    Register addressing starts at zero thus Register Number = Register Address + 1

## Modpoll example

[Modpoll](https://gavinying.github.io/modpoll/) is a comandline tool for querying ModBUS device. 

Modpoll config `evse-modpoll.csv`:
```csv
device,esp32-evse,1,,
poll,holding_register,100,12,BE_BE,
ref,state,100,string2,r
ref,error_bits,101,uint32,r
ref,charging_enabled,103,uint16,rw
ref,charging_available,104,uint16,rw
ref,pending_authorization,105,uint16,r
ref,charging_current,106,uint16,rw,A,0.1
ref,consumtion_limit,107,uint32,rw,Wh
ref,charging_time_limit,109,uint32,rw,s
ref,under_power_limit,111,uint16,rw,W
ref,authorize,112,uint16,w
poll,holding_register,200,19,BE_BE,
ref,charging_power,200,uint16,r,W
ref,session_time,201,uint32,r,s
ref,charging_time,203,uint32,r,s
ref,consumption,205,uint32,r,Wh
ref,l1_voltage,207,uint32,r,V,0.001
ref,l2_voltage,209,uint32,r,V,0.001
ref,l3_voltage,211,uint32,r,V,0.001
ref,l1_current,213,uint32,r,A,0.001
ref,l2_current,215,uint32,r,A,0.001
ref,l3_current,217,uint32,r,A,0.001
poll,holding_register,400,13,BE_BE,
ref,uptime,400,uint32,r,s
ref,low_temperature,402,int16,r,dg.C
ref,hight_temperature,403,int16,r,dg.C
ref,temperature_sensor_count,404,uint16,r
ref,app_versiov,405,string16,r
ref,reboot,421,uint16,w
```

Run with docker (set corret serial device `/dev/ttyX` and baud rate):
```sh
docker run --rm \
	--volume ./evse-modpoll.csv:/app/evse-modpoll.csv \
	--device /dev/ttyUSB1 \
	topmaker/modpoll \
	modpoll \
	--serial /dev/ttyUSB1 \
	--serial-baud 115200 \
	--config /app/evse-modpoll.csv
```
!!! note
    The docker version has the advantage that it can run everywhere and it doesn't download any other packages to your system.

Output should look like this:
```
Device: esp32-evse
+--------------------------+------------------+------+
| Reference                |            Value | Unit |
+--------------------------+------------------+------+
| state                    |                A |      |
| error_bits               |                0 |      |
| charging_enabled         |                1 |      |
| charging_available       |                1 |      |
| pending_authorization    |                0 |      |
| charging_current         |             32.0 | A    |
| consumtion_limit         |                0 | Wh   |
| charging_time_limit      |                0 | s    |
| under_power_limit        |                0 | W    |
| charging_power           |                0 | W    |
| session_time             |                0 | s    |
| charging_time            |                0 | s    |
| consumption              |                0 | Wh   |
| l1_voltage               |              0.0 | V    |
| l2_voltage               |              0.0 | V    |
| l3_voltage               |              0.0 | V    |
| l1_current               |              0.0 | A    |
| l2_current               |              0.0 | A    |
| l3_current               |              0.0 | A    |
| uptime                   |             1257 | s    |
| low_temperature          |             24.3 | dg.C |
| hight_temperature        |             24.3 | dg.C |
| temperature_sensor_count |                1 |      |
| app_versiov              | v2.1.1-35-g0fd68 |      |
+--------------------------+------------------+------+
```
