---
title: Tailscale and MagicDNS on OpenSUSE Tumbleweed
description: Getting Tailscale's MagicDNS working on OpenSUSE Tumbleweed by switching from NetworkManager's DNS handling to systemd-resolved.
date: 2025-04-03T11:55:47+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

I'm running Tailscale on my PC, my home media/backup server, and my smartphone. This allows me to have the three devices communicate with each other through a VPN.

Tailscale is installed according to its online documentation. 

To make this work well, we keep Tailscale's MagicDNS turned on. This allows me to, for example, connect to the mediaserver from my phone and PC as 'mediaserver'.  

However, by default OpenSUSE is just letting NetworkManager write to `/etc/resolve.conf` when it sets up its network connections.  Unfortunately, in this setup. Tailscale doesn't know what to do, so it does the same thing and the two end up fighting with each other.   The solution is to run systemd-resolved on OpenSUSE.  

You can read all about this fiasco here: https://tailscale.com/kb/1188/linux-dns
And if you really want to go down the rabbit hole, read this: https://tailscale.com/blog/sisyphean-dns-client-linux  (I did, and I came away with a ton of respect for the Tailscale team). 

## Solution

`sudo zypper install systemd-network`

`sudo systemctl enable systemd-resolved`

`sudo systemctl start systemd-resolved`

`sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

Now, reboot. No, really, reboot. Do not just restart NetworkManager and call it a day. Trust me.

`sudo reboot`

When `/etc/resolv.conf` is a symlink to `/run/systemd/resolve/stub-resolv.conf`, NetworkManager will automatically assume you're running systemd-resolved and will communicate with it over dbus. 

Now, when I query for 'mediaserver' it will return (and cache!) the Tailscale address if Tailscale is running. 
