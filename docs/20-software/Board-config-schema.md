---
title: Board config schema reference
---
# Board config schema

[Reference schema](http://json-schema.org/draft-07/schema#)

[Bodard schema](https://github.com/dzurikmiroslav/esp32-evse/tree/master/board-config).

# definitions

**_BoardConfig_**

 - ## Board config
 - Type: `object`
 - <i id="/definitions/BoardConfig">path: #/definitions/BoardConfig</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/BoardConfig/properties/deviceName">deviceName</b> `required`
		 - _Name of the device_
		 - Type: `string`
		 - <i id="/definitions/BoardConfig/properties/deviceName">path: #/definitions/BoardConfig/properties/deviceName</i>
		 - Length:  &le; 32
	 - <b id="#/definitions/BoardConfig/properties/ledChargingGpio">ledChargingGpio</b>
		 - _Charging led gpio number_
		 - Type: `integer`
		 - <i id="/definitions/BoardConfig/properties/ledChargingGpio">path: #/definitions/BoardConfig/properties/ledChargingGpio</i>
	 - <b id="#/definitions/BoardConfig/properties/ledErrorGpio">ledErrorGpio</b>
		 - _Error led gpio number_
		 - Type: `integer`
		 - <i id="/definitions/BoardConfig/properties/ledErrorGpio">path: #/definitions/BoardConfig/properties/ledErrorGpio</i>
	 - <b id="#/definitions/BoardConfig/properties/ledWifiGpio">ledWifiGpio</b>
		 - _WiFi led gpio number_
		 - Type: `integer`
		 - <i id="/definitions/BoardConfig/properties/ledWifiGpio">path: #/definitions/BoardConfig/properties/ledWifiGpio</i>
	 - <b id="#/definitions/BoardConfig/properties/buttonGpio">buttonGpio</b>
		 - _Button gpio number_
		 - Type: `integer`
		 - <i id="/definitions/BoardConfig/properties/buttonGpio">path: #/definitions/BoardConfig/properties/buttonGpio</i>
	 - <b id="#/definitions/BoardConfig/properties/pilot">pilot</b> `required`
		 - <i id="/definitions/BoardConfig/properties/pilot">path: #/definitions/BoardConfig/properties/pilot</i>
		 - &#36;ref: [#/definitions/Pilot](#/definitions/Pilot)
	 - <b id="#/definitions/BoardConfig/properties/proximity">proximity</b>
		 - <i id="/definitions/BoardConfig/properties/proximity">path: #/definitions/BoardConfig/properties/proximity</i>
		 - &#36;ref: [#/definitions/Proximity](#/definitions/Proximity)
	 - <b id="#/definitions/BoardConfig/properties/acRelayGpio">acRelayGpio</b> `required`
		 - _AC relay gpio number for controlling mains contactor_
		 - Type: `integer`
		 - <i id="/definitions/BoardConfig/properties/acRelayGpio">path: #/definitions/BoardConfig/properties/acRelayGpio</i>
	 - <b id="#/definitions/BoardConfig/properties/socketLock">socketLock</b>
		 - <i id="/definitions/BoardConfig/properties/socketLock">path: #/definitions/BoardConfig/properties/socketLock</i>
		 - &#36;ref: [#/definitions/SocketLock](#/definitions/SocketLock)
	 - <b id="#/definitions/BoardConfig/properties/rcm">rcm</b>
		 - <i id="/definitions/BoardConfig/properties/rcm">path: #/definitions/BoardConfig/properties/rcm</i>
		 - &#36;ref: [#/definitions/Rcm](#/definitions/Rcm)
	 - <b id="#/definitions/BoardConfig/properties/energyMeter">energyMeter</b>
		 - <i id="/definitions/BoardConfig/properties/energyMeter">path: #/definitions/BoardConfig/properties/energyMeter</i>
		 - &#36;ref: [#/definitions/EnergyMeter](#/definitions/EnergyMeter)
	 - <b id="#/definitions/BoardConfig/properties/auxInputs">auxInputs</b>
		 - _Aux inputs configuration_
		 - Type: `array`
		 - <i id="/definitions/BoardConfig/properties/auxInputs">path: #/definitions/BoardConfig/properties/auxInputs</i>
			 - **_Items_**
			 - <i id="/definitions/BoardConfig/properties/auxInputs/items">path: #/definitions/BoardConfig/properties/auxInputs/items</i>
			 - &#36;ref: [#/definitions/AuxInput](#/definitions/AuxInput)
	 - <b id="#/definitions/BoardConfig/properties/auxOutputs">auxOutputs</b>
		 - _Aux outputs configuration_
		 - Type: `array`
		 - <i id="/definitions/BoardConfig/properties/auxOutputs">path: #/definitions/BoardConfig/properties/auxOutputs</i>
			 - **_Items_**
			 - <i id="/definitions/BoardConfig/properties/auxOutputs/items">path: #/definitions/BoardConfig/properties/auxOutputs/items</i>
			 - &#36;ref: [#/definitions/AuxOutput](#/definitions/AuxOutput)
	 - <b id="#/definitions/BoardConfig/properties/auxAnalogInputs">auxAnalogInputs</b>
		 - _Aux analog inputs configuration_
		 - Type: `array`
		 - <i id="/definitions/BoardConfig/properties/auxAnalogInputs">path: #/definitions/BoardConfig/properties/auxAnalogInputs</i>
			 - **_Items_**
			 - <i id="/definitions/BoardConfig/properties/auxAnalogInputs/items">path: #/definitions/BoardConfig/properties/auxAnalogInputs/items</i>
			 - &#36;ref: [#/definitions/AuxAnalogInput](#/definitions/AuxAnalogInput)
	 - <b id="#/definitions/BoardConfig/properties/serials">serials</b>
		 - _Serial ports configuration_
		 - Type: `array`
		 - <i id="/definitions/BoardConfig/properties/serials">path: #/definitions/BoardConfig/properties/serials</i>
		 - Item Count:  &le; 3
			 - **_Items_**
			 - <i id="/definitions/BoardConfig/properties/serials/items">path: #/definitions/BoardConfig/properties/serials/items</i>
			 - &#36;ref: [#/definitions/Serial](#/definitions/Serial)
	 - <b id="#/definitions/BoardConfig/properties/onewire">onewire</b>
		 - <i id="/definitions/BoardConfig/properties/onewire">path: #/definitions/BoardConfig/properties/onewire</i>
		 - &#36;ref: [#/definitions/Onewire](#/definitions/Onewire)


**_EnergyMeter_**

 - ## EnergyMeter
 - _Energy meter configuration_
 - Type: `object`
 - <i id="/definitions/EnergyMeter">path: #/definitions/EnergyMeter</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/EnergyMeter/properties/currentAdcChannels">currentAdcChannels</b> `required`
		 - _Current sensing of L1, L2, L3 adc1 channels, for single phase set array with one item_
		 - Type: `array`
		 - <i id="/definitions/EnergyMeter/properties/currentAdcChannels">path: #/definitions/EnergyMeter/properties/currentAdcChannels</i>
		 - Item Count:  &le; 3
			 - **_Items_**
			 - Type: `integer`
			 - <i id="/definitions/EnergyMeter/properties/currentAdcChannels/items">path: #/definitions/EnergyMeter/properties/currentAdcChannels/items</i>
	 - <b id="#/definitions/EnergyMeter/properties/currentScale">currentScale</b> `required`
		 - _Multiplier of measured current values_
		 - Type: `number`
		 - <i id="/definitions/EnergyMeter/properties/currentScale">path: #/definitions/EnergyMeter/properties/currentScale</i>
	 - <b id="#/definitions/EnergyMeter/properties/voltageAdcChannels">voltageAdcChannels</b>
		 - _Voltage sensing of L1, L2, L3 adc1 channels, for single phase set array with one item_
		 - Type: `array`
		 - <i id="/definitions/EnergyMeter/properties/voltageAdcChannels">path: #/definitions/EnergyMeter/properties/voltageAdcChannels</i>
		 - Item Count:  &le; 3
			 - **_Items_**
			 - Type: `integer`
			 - <i id="/definitions/EnergyMeter/properties/voltageAdcChannels/items">path: #/definitions/EnergyMeter/properties/voltageAdcChannels/items</i>
	 - <b id="#/definitions/EnergyMeter/properties/voltageScale">voltageScale</b>
		 - _Multiplier of measured voltage values_
		 - Type: `number`
		 - <i id="/definitions/EnergyMeter/properties/voltageScale">path: #/definitions/EnergyMeter/properties/voltageScale</i>


**_Pilot_**

 - ## Pilot
 - _Control pilot configuration_
 - Type: `object`
 - <i id="/definitions/Pilot">path: #/definitions/Pilot</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/Pilot/properties/gpio">gpio</b> `required`
		 - _Generating pwm gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Pilot/properties/gpio">path: #/definitions/Pilot/properties/gpio</i>
	 - <b id="#/definitions/Pilot/properties/adcChannel">adcChannel</b> `required`
		 - _Measuring adc1 channel_
		 - Type: `integer`
		 - <i id="/definitions/Pilot/properties/adcChannel">path: #/definitions/Pilot/properties/adcChannel</i>
	 - <b id="#/definitions/Pilot/properties/levels">levels</b> `required`
		 - _Measured threshold values for voltages 12, 9, 6, 3 -12 in mV_
		 - Type: `array`
		 - <i id="/definitions/Pilot/properties/levels">path: #/definitions/Pilot/properties/levels</i>
		 - Item Count: between 5 and 5
			 - **_Items_**
			 - Type: `integer`
			 - <i id="/definitions/Pilot/properties/levels/items">path: #/definitions/Pilot/properties/levels/items</i>


**_Proximity_**

 - ## Proximity
 - _Proximity pilot configuration_
 - Type: `object`
 - <i id="/definitions/Proximity">path: #/definitions/Proximity</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/Proximity/properties/adcChannel">adcChannel</b> `required`
		 - _Measuring adc1 channel_
		 - Type: `integer`
		 - <i id="/definitions/Proximity/properties/adcChannel">path: #/definitions/Proximity/properties/adcChannel</i>
	 - <b id="#/definitions/Proximity/properties/levels">levels</b> `required`
		 - _Measured threshold values for cable 13A, 20A, 32A in mV_
		 - Type: `array`
		 - <i id="/definitions/Proximity/properties/levels">path: #/definitions/Proximity/properties/levels</i>
		 - Item Count: between 3 and 3
			 - **_Items_**
			 - Type: `integer`
			 - <i id="/definitions/Proximity/properties/levels/items">path: #/definitions/Proximity/properties/levels/items</i>


**_SocketLock_**

 - ## SocketLock
 - _Socket lock configuration_
 - Type: `object`
 - <i id="/definitions/SocketLock">path: #/definitions/SocketLock</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/SocketLock/properties/aGpio">aGpio</b> `required`
		 - _Lock A gpio number_
		 - Type: `integer`
		 - <i id="/definitions/SocketLock/properties/aGpio">path: #/definitions/SocketLock/properties/aGpio</i>
	 - <b id="#/definitions/SocketLock/properties/bGpio">bGpio</b> `required`
		 - _Lock B gpio number_
		 - Type: `integer`
		 - <i id="/definitions/SocketLock/properties/bGpio">path: #/definitions/SocketLock/properties/bGpio</i>
	 - <b id="#/definitions/SocketLock/properties/detectionGpio">detectionGpio</b> `required`
		 - _Detection gpio number_
		 - Type: `integer`
		 - <i id="/definitions/SocketLock/properties/detectionGpio">path: #/definitions/SocketLock/properties/detectionGpio</i>
	 - <b id="#/definitions/SocketLock/properties/detectionDelay">detectionDelay</b> `required`
		 - _Delay after locking/unlocking for check state in ms_
		 - Type: `integer`
		 - <i id="/definitions/SocketLock/properties/detectionDelay">path: #/definitions/SocketLock/properties/detectionDelay</i>
	 - <b id="#/definitions/SocketLock/properties/minBreakTime">minBreakTime</b> `required`
		 - _Min break time for repeated locking/unlocking in ms_
		 - Type: `integer`
		 - <i id="/definitions/SocketLock/properties/minBreakTime">path: #/definitions/SocketLock/properties/minBreakTime</i>


**_Rcm_**

 - ## Rcm
 - _Residual current monitor configuration_
 - Type: `object`
 - <i id="/definitions/Rcm">path: #/definitions/Rcm</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/Rcm/properties/gpio">gpio</b> `required`
		 - _Sensing gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Rcm/properties/gpio">path: #/definitions/Rcm/properties/gpio</i>
	 - <b id="#/definitions/Rcm/properties/testGpio">testGpio</b> `required`
		 - _Test gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Rcm/properties/testGpio">path: #/definitions/Rcm/properties/testGpio</i>


**_AuxInput_**

 - ## Aux input
 - _Aux input configuration_
 - Type: `object`
 - <i id="/definitions/AuxInput">path: #/definitions/AuxInput</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/AuxInput/properties/name">name</b> `required`
		 - _Name_
		 - Type: `string`
		 - <i id="/definitions/AuxInput/properties/name">path: #/definitions/AuxInput/properties/name</i>
		 - Length:  &le; 8
	 - <b id="#/definitions/AuxInput/properties/gpio">gpio</b> `required`
		 - _The gpio number_
		 - Type: `integer`
		 - <i id="/definitions/AuxInput/properties/gpio">path: #/definitions/AuxInput/properties/gpio</i>


**_AuxOutput_**

 - ## Aux output
 - _Aux output configuration_
 - Type: `object`
 - <i id="/definitions/AuxOutput">path: #/definitions/AuxOutput</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/AuxOutput/properties/name">name</b> `required`
		 - _Name_
		 - Type: `string`
		 - <i id="/definitions/AuxOutput/properties/name">path: #/definitions/AuxOutput/properties/name</i>
		 - Length:  &le; 8
	 - <b id="#/definitions/AuxOutput/properties/gpio">gpio</b> `required`
		 - _The gpio number_
		 - Type: `integer`
		 - <i id="/definitions/AuxOutput/properties/gpio">path: #/definitions/AuxOutput/properties/gpio</i>


**_AuxAnalogInput_**

 - ## Aux analog input
 - _Aux analog input configuration_
 - Type: `object`
 - <i id="/definitions/AuxAnalogInput">path: #/definitions/AuxAnalogInput</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/AuxAnalogInput/properties/name">name</b> `required`
		 - _Name_
		 - Type: `string`
		 - <i id="/definitions/AuxAnalogInput/properties/name">path: #/definitions/AuxAnalogInput/properties/name</i>
		 - Length:  &le; 8
	 - <b id="#/definitions/AuxAnalogInput/properties/adcChannel">adcChannel</b> `required`
		 - _The adc1 channel_
		 - Type: `integer`
		 - <i id="/definitions/AuxAnalogInput/properties/adcChannel">path: #/definitions/AuxAnalogInput/properties/adcChannel</i>


**_Serial_**

 - ## Serial
 - _Serial port configuration_
 - Type: `object`
 - <i id="/definitions/Serial">path: #/definitions/Serial</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/Serial/properties/type">type</b>
		 - _Type_
		 - Type: `string`
		 - <i id="/definitions/Serial/properties/type">path: #/definitions/Serial/properties/type</i>
		 - The value is restricted to the following: 
			 1. _"uart"_
			 2. _"rs485"_
	 - <b id="#/definitions/Serial/properties/name">name</b>
		 - _Name_
		 - Type: `string`
		 - <i id="/definitions/Serial/properties/name">path: #/definitions/Serial/properties/name</i>
		 - Length:  &le; 16
	 - <b id="#/definitions/Serial/properties/rxdGpio">rxdGpio</b>
		 - _RX data gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Serial/properties/rxdGpio">path: #/definitions/Serial/properties/rxdGpio</i>
	 - <b id="#/definitions/Serial/properties/txdGpio">txdGpio</b>
		 - _TX data gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Serial/properties/txdGpio">path: #/definitions/Serial/properties/txdGpio</i>
	 - <b id="#/definitions/Serial/properties/rtsGpio">rtsGpio</b>
		 - _Flow control signal gpio number, required for rs485_
		 - Type: `integer`
		 - <i id="/definitions/Serial/properties/rtsGpio">path: #/definitions/Serial/properties/rtsGpio</i>


**_Onewire_**

 - ## Onewire
 - _Onewire bus configuration_
 - Type: `object`
 - <i id="/definitions/Onewire">path: #/definitions/Onewire</i>
 - This schema <u>does not</u> accept additional properties.
 - **_Properties_**
	 - <b id="#/definitions/Onewire/properties/gpio">gpio</b> `required`
		 - _Onewire bus gpio number_
		 - Type: `integer`
		 - <i id="/definitions/Onewire/properties/gpio">path: #/definitions/Onewire/properties/gpio</i>
	 - <b id="#/definitions/Onewire/properties/temperatureSensor">temperatureSensor</b> `required`
		 - _Has temperature sensor on onewire bus_
		 - Type: `boolean`
		 - <i id="/definitions/Onewire/properties/temperatureSensor">path: #/definitions/Onewire/properties/temperatureSensor</i>

_Generated with [json-schema-md-doc](https://brianwendt.github.io/json-schema-md-doc/)_