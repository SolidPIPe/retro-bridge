<br />
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/retro-bridge-logo-light.svg">
    <img alt="Retro-Bridge Logo" src="assets/retro-bridge-logo-dark.svg" width="500" style="max-width: 100%;">
  </picture>
</p>

<p align="center">Connect to your RetroTINK wirelessly</p>

<p align="center">
  <a href="https://github.com/solidpipe/retro-bridge/releases/latest">
    <img src="https://img.shields.io/github/v/release/solidpipe/retro-bridge?label=Latest%20Version&color=blue" alt="Latest Version">
  </a>
<br /> <br />
  <a href="https://ko-fi.com/solidpipe">
  <img src="https://storage.ko-fi.com/cdn/brandasset/v2/support_me_on_kofi_beige.png" alt="Support me on Ko-fi" width="200">
</a>
</p>

# Overview
Retro-Bridge connects your devices to your RetroTINK wirelessly. It serves as a Wi-Fi bridge for serial communication with RetroTINK devices, without the need for physical cables.

Use apps like [RetroTINK Remote](https://rt4k-remote.pipe.hr/) and [RetroTINK Profiler](https://rt4k-profiler.pipe.hr/) from any device. Send remote commands, change settings live, update the firmware, manage files on the SD card, and more!

Developer documentation for the Retro-Bridge protocol will be released soon. Build your own integrations, automations, and fun projects with your RetroTINK (with no wires in sight!).

Retro-Bridge is based on a Raspberry Pi Pico 2 W, supports 2.4GHz Wi-Fi, and connects to your RetroTINK via USB or HD-15.

<br />

<p align="center">
<img src="assets/retro-bridge-dashboard.png" alt="Retro-Bridge Dashboard">
</p>

# Updating Retro-Bridge
Updating the Retro-Bridge is easy. Simply follow the [installation instructions steps 1-4](#installation-instructions-usb). Retro-Bridge will remember your Wi-Fi credentials between firmware updates.

# Setup

## Recommended: USB Connection (4K Pro/CE)
Connecting Retro-Bridge to the RetroTINK via USB is the easiest and the most performant connection method. This is likely how you'll be using Retro-Bridge.

You will need the following:
- [Raspberry Pi Pico 2 W](https://amzn.to/3SLW5LT) (must be a Pico 2 W. Retro-Bridge won't work on other models)
- Micro USB cable
- [USB-C OTG adapter](https://amzn.to/4yG8znu) (this one has been working well)

Some USB-C OTG adapters may be incompatible. Buy from a retailer where you can return them easily if needed. 

### Installation Instructions (USB):
1. Download the latest version of Retro-Bridge from the [Releases page](https://github.com/solidpipe/retro-bridge/releases/latest). 
2. Hold the BOOTSEL button on the Pico as you're plugging the Micro USB to your computer. It should open as a drive in your Explorer.
3. Copy the `.uf2` file to the root of that drive. After the copy, the drive will disappear. You can unplug the Pico from the computer now.
4. Plug the Pico to the USB-C OTG adapter connected to the Tink. Connect the USB-C power supply to the USB-C OTG adapter. With everything powered on, the Pico will blink the LED.
5. Connect to the Wi-Fi network called "Retro-Bridge-XXXX". A captive portal page should open after about ~10-15 seconds (if it doesn't, visit http://192.168.4.1).
6. Select your Wi-Fi network and enter your password. Hit Connect. If successful, the Pico will connect to your Wi-Fi and the captive portal should close on its own.
7. On your computer or phone, visit http://retro-bridge.local/. If you see the Retro-Bridge dashboard, the setup was successful!

## HD-15 Connection (6X CE)
⚠️ Note: HD-15 Serial connection requires advanced understanding of electronics. Miswiring may permanently damage your RetroTINK, the Pico, or both. Use at your own risk. ⚠️

**I'm working on plug & play HD-15 Retro-Bridge boards. They're currently in prototype stage, and I'm hoping to have them available for purchase soon.**

RetroTINK devices expose serial connection over the HD-15 port on pins 12 (TX) and 15 (RX). Retro-Bridge uses GP8 (Pico pin 11) as TX and GP9 (Pico pin 12) as RX. 
<br />⚠️ Important: You will need to add pull-up resistors between 3.3V Out (Pin 36) and pins 12 and 15. For more information, check out the RetroTINK Wiki: https://consolemods.org/wiki/AV:RetroTINK-4K#Serial_Over_USB_/_HD-15

### Installation Instructions (HD-15):
1. Download the latest version of Retro-Bridge from the [Releases page](https://github.com/solidpipe/retro-bridge/releases/latest). 
2. Make sure no HD-15 cables between the Pico and the Tink are connected.
3. Hold the BOOTSEL button on the Pico as you're plugging the Micro USB to your computer. It should open as a drive in your Explorer.
4. Copy the `.uf2` file to the root of that drive. After the copy, the drive will disappear. You can unplug the Pico from the computer now.
5. Plug the Pico to USB power or the USB-C OTG adapter connected to the Tink. With everything powered on, the Pico will blink the LED.
6. Connect the HD-15 cable between the Pico and the Tink.
7. Connect to the Wi-Fi network called "Retro-Bridge-XXXX". A captive portal page should open after about ~10-15 seconds (if it doesn't, visit http://192.168.4.1).
8. Select your Wi-Fi network and enter your password. Hit Connect. If successful, the Pico will connect to your Wi-Fi and the captive portal should close on its own.
9. On your computer or phone, visit http://retro-bridge.local/. If you see the Retro-Bridge dashboard, the setup was successful!


## Hardware Compatibility

| Model | USB | HD-15 | Note |
| :--- | :---: | :---: | --- |
| RetroTINK 4K Pro | ✅  | ✅ |
| RetroTINK 4K CE | ✅ | ✅ |
| RetroTINK 6X CE | ❌ | ✅ | 6X CE doesn't support serial over USB. Compatible Retro-Bridge HD-15 boards will be available in the near future

<sub>*RetroTINK 5X, and 2X are not supported.</sub>

# Troubleshooting

I can't connect Retro-Bridge to my Wi-Fi, or my Wi-Fi connection is unstable:
- Make sure you're connecting to a 2.4GHz Wi-Fi connection. The Pico 2 W does not support 5GHz Wi-Fi.
- Use WPA2 (recommended) or WPA2/3 Mixed mode networks. Pure WPA3 networks are not recommended.
- Check the signal strength on the [Retro-Bridge Dashboard](http://retro-bridge.local/). Move the device closer to your Wi-Fi access point and ensure no (metal) objects are surrounding the Pico.

I can't load http://retro-bridge.local/:
- Make sure mDNS is allowed on your network
- Alternatively, check your Wi-Fi Gateway's client list to find your Retro-Bridge's IP address. 

## Changing Wi-Fi settings

If you'd like to change your Wi-Fi settings, hold the BOOTSEL button on the Pico for ~8 seconds (when the LED starts blinking, you can release the button). This will put Retro-Bridge in recovery mode, from which you can select a different Wi-Fi network.




