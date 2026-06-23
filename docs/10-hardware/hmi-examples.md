---
title: HMI examples
---

A couple of options to create Human-Machine Interface for ESP32-EVSE.

## Nextion

The [Nextion API](../20-software/Nextion.md) built into the firmware allows implementation of Nextion displays as HMI. 

![Nextion](/images/hmi-nextion.gif)

For an advanced example of HMI built with a Nextion display check out the [esp32-evse-nextion](https://github.com/dzurikmiroslav/esp32-evse-nextion) repository.

## ESPHome with LVGL

The [AT commands engine](../20-software/AT-Commands.md) can be used to communicate from other open source projects like ESPHome with ESP32-EVSE. 

This example relies on the [external ESPHome component](https://github.com/nagyrobi/esp32evse-esphome) to integrate communication, and supports several display types. Check out the [esp32evse_esphome-lvgl](https://github.com/nagyrobi/esp32evse_esphome-lvgl) repository for details.

![ESPHome](/images/hmi-esphome-lvgl.gif)

This can be used not only for graphical displays. One may implement [just a LED strip with a pixel animation to suggest charging state](https://github.com/nagyrobi/esp32evse_esphome-lvgl/blob/main/esp32-evse_ledstrip_for_twc2.yaml).

, 