---
title: Snap on OpenSUSE Tumbleweed
description: 
date: 2025-04-03T15:22:12+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

Well, this was fun.  I don't use Snap for a lot, but Obsidian is an exception, so here I am, installing Snap.  Well, good news: Snapcraft has instructions for just my scenario!

https://snapcraft.io/install/obsidian/opensuse

But, there's a catch. 

Sometime in 2024/2025 OpenSUSE switched to SELinux by default, away from AppArmor. Unfortunately as of early 2025 AppArmor is required to run `snapd`. 

## Solution

For the time being, I'm not messing with SELinux.  I switched back to AppArmor via kernel flags:

In 

`/etc/default/grub` 

remove `selinux=1` and `security=selinux` and replace with `security=apparmor`.  

Then `sudo grub2-mkconfig -o /boot/efi/EFI/opensuse/grub.cfg`

and reboot:

`sudo reboot`

Now, the instructions on snapcraft.io, which include enabling a `snapd.apparmor` systemd service, actually make sense and work. 

## Follow-Up

I'm not running snapd on OpenSUSE anymore.  The only reason I was using it was for Obsidian, and that was to avoid having AppImages on my system. I've since started using appimaged which makes it much, much, easier to download, install, and update AppImages so I have Obsidian installed that way now.