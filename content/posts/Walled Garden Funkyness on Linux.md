---
title: "Walled Garden Funkyness on Linux"
description: 
date: 2026-01-23T09:04:44-05:00
draft: false
tags:
  - linux
showTags: true
---

# Background

Most days I work at my little office in town. It's a nice, quiet space I share with a friend of mine who's a psychotherapist. We each have closed-door offices, we get along well, and the fiber-optic gigabit is pretty plum.  But, some days I need to break the cycle, so I go to a co-working space run by another family friend. 

At the co-working space, Internet access is provided by a non-encrypted network. The experience is a classic walled garden: you connect your device to the public network, you try to access something on the web and it redirects you to the internal login page (which reveals a Ruckus Networks controller at the helm). At this place all that means is you click on an "I agree to the terms" checkbox and a Submit button, and now you're online.  

If you're a reader of my posts you'll know I run Linux on the desktop. Specifically OpenSUSE Tumbleweed. And if you have any experience with Linux on the desktop you'll know what I'm about to say:  this thing works great on Macs, Windows PCs, Android and iOS devices... but not my Linux desktop. 

Since we're a quarter of the way through the 21st century, you would think that this would not still be the case but alas.

## What happens

The first time I ever visited this site with this machine, everything worked great. I selected the network from my available wireless networks. I was prompted to "Log In" to the network by KDE Plasma's network widget. This fires up http://conncheck.opensuse.org in a web browser, which redirects me to the walled garden page. I check the box, I click submit and I'm online.

Subsequent visits don't go so well. My PC remembers the network and reconnects to it automatically. I may (or may not) be prompted with the 'Login In' toast, depending on how long it's been since I connected.  If I am prompted, clicking it will attempt to load conncheck.opensuse.org in a browser, but the redirection will never occur. I will never see the walled garden page. 

## What does work

If this is happening to you, there's a quick fix:

1. Stop NetworkManager
2. Turn off wireless.
3. Delete the connection profile for this particular wireless network.
4. Restart NetworkManager
5. Turn wireless back on
6. Click on the wireless network and set it up fresh. You should get the "Log In" toast and this time conncheck.opensuse.org should redirect you to the walled garden


## What actually happened?

Modern NetworkManager (which openSUSE Tumbleweed uses) randomizes your MAC address for privacy reasons. The default behavior varies by distro and version, but here's the sequence causing your problem:

1. **First visit**: NetworkManager assigns a random MAC (let's call it `AA:BB:CC:11:22:33`). You authenticate through the captive portal. The Ruckus controller records: "MAC `AA:BB:CC:11:22:33` has agreed to terms, allow traffic."
2. **Next visit (days/weeks later)**: NetworkManager reconnects to the saved network profile, but generates a _different_ random MAC (`AA:BB:CC:44:55:66`). The Ruckus controller has never seen this MAC before—you're unauthenticated.
3. **The failure**: NetworkManager sees a _known_ network profile and assumes things should "just work." NetworkManager tries to reach a webserver after connecting to detect if it's behind a captive portal [ArchWiki](https://wiki.archlinux.org/title/NetworkManager), but there's a timing/state mismatch. The captive portal detection either doesn't trigger properly, or triggers but the redirect fails because NetworkManager is in a confused state about this "known" network.
4. **Why the workaround works**: Deleting the profile forces NetworkManager to treat it as a completely new network, running fresh captive portal detection from scratch.

## The Better Fix

Set the MAC address mode to **`stable`** for this specific network. The "stable" method generates a stable, hashed value—every time the connection activates, the same address is generated. This is useful so that a captive-portal might remember your login-status based on the MAC address.

Run this command (replace `CoworkingSpaceName` with the actual SSID):

```bash
nmcli connection modify "CoworkingSpaceName" wifi.cloned-mac-address stable
```

Now every time you connect to that network, you'll present the same pseudo-random MAC, the Ruckus controller will recognize you, and you should sail through (at least until the grace period expires—Ruckus has a grace period which sets the number of minutes during which previously authenticated clients that disconnect can rejoin without going through authentication again.

## Why This Mostly Affects Linux

Windows, macOS, iOS, and Android all have MAC randomization too, but they've had years of refinement around captive portal edge cases. With `random` you may be required to re-authenticate (or click "I agree") on every connect. Other OSes tend to default to `stable`-like behavior for saved networks, while Linux distros have been more aggressive about privacy defaults without always handling the captive portal interaction gracefully.