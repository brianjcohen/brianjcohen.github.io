---
title: Fixing Flameshot on KDE Plasma with dual monitors under Wayland
description: 
date: 2025-04-23T09:29:52+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

There are no shortage of reported issues with [Flameshot](https://flameshot.org) under Wayland.  Interestingly, they have an official [Wayland Help Page](https://flameshot.org/docs/guide/wayland-help/) that doesn't address most of the concerns you see showing up in their [github](https://github.com/flameshot-org/flameshot/issues?q=is%3Aissue%20wayland). 

In my setup, I have two monitors side-by-side. I'm running Wayland on OpenSUSE with KDE Plasma.

Here's what would happen to me:

1. Flameshot is installed via my OS's package management system (OpenSUSE zypper).
2. Flameshot launches to the system tray in my panel, as it should. 
3. Click the Flameshot launcher.  The shaded overlay appears but only on one monitor, and only allowing me to select a portion of one monitor for capture (and they're not always the same!)

## The Fix

The fix is buried deep in this github Issue:
https://github.com/flameshot-org/flameshot/issues/2364#issuecomment-2845331480

1. Load up System Settings, Autostart
2. Flameshot should be in the list. Click the little icon directly to the left of the X in its row.
3. Go to the Application tab.
4. Add QT_QPA_PLATFORM=xcb to the Environment Variables line
5. Click OK.
6. Log out and back in.

That's it. This will force Flameshot to use XWayland and resolves the dual-monitor weirdness.


