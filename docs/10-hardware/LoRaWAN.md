For communication over LoRaWAN network I used Dragino RS485-LN (Modbus to LoRaWAN Converter).

## EVSE configuration

Set RS-485 serial interface type for modbus mode, and baud rate: 9600, data bits: 8, stop bits: 1, parity: disable.
(You can set your own configuration parameters, but don't forget to set equal: AT+BAUDR, AT+PARITY, AT+DATABIT, AT+STOPBIT settings in Dragino)

## Dragino configuration

```AT
AT+TDC=60000
AT+DECRYPT=0
AT+MBFUN=0
AT+BAUDR=9600
AT+PARITY=0
AT+DATABIT=8
AT+STOPBIT=0
AT+DATAUP=0
AT+PAYVER=1
AT+CHS=0
AT+RXMODE=0,0
AT+COMMAND1=01 03 00 64 00 04, 1
AT+DATACUT1=13,2,4~9+11~11
AT+CMDDL1=0
AT+COMMAND2=01 03 00 c8 00 03, 1
AT+DATACUT2=11,2,4~9
AT+CMDDL2=0
AT+COMMAND3=01 03 00 cf 00 0c, 1
AT+DATACUT3=29,2,4~27
AT+CMDDL3=0
AT+COMMAND4=01 03 01 90 00 04, 1
AT+DATACUT4=13,2,4~7+10~11
AT+CMDDL4=0
```

## The Things Network

After sucesffuly added Dragino to your app, set JavaScript payload decoder:

```js
function decodeUplink(input) {
  function uint32(idx) {
   return input.bytes[idx] << 24 | input.bytes[idx + 1] << 16 | input.bytes[idx + 2] << 8 | input.bytes[idx + 3];
  }
  
  function uint16(idx) {
   return input.bytes[idx] << 8 | input.bytes[idx + 1];
  }
  
  function toSint16(value) {
    if ((value & 1 << 15) > 0) { // value is negative (16bit 2's complement)
      value = ((~value) & 0xffff) + 1; // invert 16bits & add 1 => now positive value
      value = value * -1;
    }
    return value;
  }
  
  return {
    data: {
      state: String.fromCharCode(input.bytes[1]) + (input.bytes[2] > 0 ? String.fromCharCode(input.bytes[2]) : ''),
      errors: uint32(3),
      enabled: input.bytes[7] === 1,
      power: uint16(8),
      consumption: uint32(10),
      voltage: [
        uint32(14) / 1000,
        uint32(18) / 1000,
        uint32(22) / 1000,
      ],
      current: [
        uint32(26) / 1000,
        uint32(30) / 1000,
        uint32(34) / 1000,
      ],
      uptime: uint32(38),
      temperature: toSint16(uint16(42)) / 100,
    },
    warnings: [],
    errors: []
  };
}
```

After setup, you should see live data in TTN console like this:

![TTN live data](/images/ttn-live-data.png "TTN live data")