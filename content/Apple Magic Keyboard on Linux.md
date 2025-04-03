---
title: "Apple Magic Keyboard on Linux"
description: 
date: "2025-04-03T11:47:06+00:00"
draft: false
---
## Background

I have mixed feelings about Apple products in general, but I'm all-in on their keyboards and have been for 20 years.  I loved the old wired Aluminum keyboards but unfortunately they don't make 'em anymore.  When I needed a new one, I bought the Bluetooth "Magic Keyboard" they now sell in its place. 

Unfortunately, virtually no online documentation for this keyboard is useful; it’s all Mac-specific. 

## Solution

To pair (or re-pair) this device with a Linux machine:

1. Turn the hardware switch off (it's up near the number paid on the front edge)
2. Make sure the Lightning cable is unplugged. 
3. Go to your system's Bluetooth settings and click Add a Device (at least that's what it's called in KDE Plasma)
4. Turn the switch on.  The keyboard should appear in the list of available devices.  
5. Select the device for pairing. It will prompt you to confirm a PIN, just say Yes (obviously the keyboard is not prompting you with a PIN!)

As described [here](https://forums.opensuse.org/t/login-with-bluetooth-keyboard-sddm-kde-plasma/145506/10), on some distributions (including OpenSUSE Tumbleweed), out of the box bluetooth keyboards will only connect via bluetooth after you login. This means you wouldn’t be able to use your bluetooth keyboard within the login screen (GDM or SDDM) upon initial boot. To fix this create the file `/etc/bluetooth/main.conf` containing:
   

```
[Policy]
AutoEnable=true
```
