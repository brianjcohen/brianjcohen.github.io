---
title: Sleep and Hibernate on OpenSUSE Tumbleweed with KDE Plasma
description: Getting reliable sleep, hibernate, and sleep-then-hibernate working on OpenSUSE Tumbleweed with the right kernel parameters, polkit rules, and KDE power settings.
date: 2026-02-10T11:00:00-05:00
draft: false
tags:
  - linux
showTags: true
---
## Background

In general, sleep does not work very well on Linux. The battery drains fast and there are often issues where things don't behave properly after waking. Hibernate is much more stable, but depending on your distribution it has to be enabled manually.

What I really wanted was sleep-then-hibernate: close my laptop lid, it goes to sleep immediately so I can change venues (leave the office, come home, go back to work after dinner), and if I don't open it again within a couple hours, it hibernates to save the battery overnight or over the weekend. macOS and Windows have been doing this for years. On OpenSUSE Tumbleweed, it took some work.

## Prerequisites

You need swap space at least as large as your RAM for hibernate to work. If you're installing OpenSUSE from scratch, choose "Guided Install" and make sure you configure the partitions to include a swap partition. If you already did this, skip ahead to Kernel Parameters.

### Adding swap after the fact

If you're already installed without swap, you have two options.

**If you're using LVM** (which the OpenSUSE guided installer sets up by default), you can carve out a logical volume from your existing volume group. First, check your volume group name and available space:

```bash
sudo vgs
```

Then create the swap volume, format it, and enable it. Replace `64G` with however much RAM you have:

```bash
sudo lvcreate -L 64G -n swap system
sudo mkswap /dev/system/swap
sudo swapon /dev/system/swap
```

Make it permanent by adding this line to `/etc/fstab`:

```
/dev/system/swap  none  swap  defaults  0  0
```

Your `resume=` kernel parameter (covered below) will point at `/dev/system/swap`.

**If you're not using LVM**, you can create a swap file instead. This works fine for sleep, but hibernate with a swap file requires an extra step — the kernel needs to know the physical offset of the file on disk.

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=65536 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Add to `/etc/fstab`:

```
/swapfile  none  swap  defaults  0  0
```

Now find the physical offset for the `resume_offset=` kernel parameter:

```bash
sudo filefrag -v /swapfile | head -4
```

Look at the first row under `physical_offset` — that's the number you need. Your kernel parameters will need both `resume=/dev/sdXY` (whatever partition the swapfile lives on) and `resume_offset=<that number>`.

Honestly, if you have the choice, LVM is much less of a headache here.

## Kernel parameters

In `/etc/default/grub`, my `GRUB_CMDLINE_LINUX_DEFAULT` looks like this:

```
splash=silent nvme.noacpi=1 acpi_osi='Windows 2020' mem_sleep_default=deep resume=/dev/system/swap mitigations=auto quiet security=apparmor
```

The important bits for sleep and hibernate:

- **`mem_sleep_default=deep`** enables S3 (deep) sleep. Without this, the kernel may default to s2idle which is shallower and drains more battery.

- **`resume=/dev/system/swap`** tells the kernel where to find the hibernation image on resume. Point this at your swap partition.

- **`nvme.noacpi=1`** tells the NVMe driver not to apply ACPI quirks. This helps with D3 suspend/resume because it tells the driver to request suspend in the standard way, rather than applying workarounds for hardware that doesn't actually need them. More detail [here](https://www.reddit.com/r/archlinux/comments/12abf5e/what_does_nvmenoacpi1_do/).

- **`acpi_osi='Windows 2020'`** tells the BIOS that we're a recent version of Windows, which can unlock power management features that the BIOS might otherwise hide from Linux. A good explanation of this flag is on the [Manjaro forum](https://forum.manjaro.org/t/how-to-choose-the-proper-acpi-kernel-argument/1405), although honestly I think a lot of people are just copying this from each other without questioning it, and there's a good chance we don't need it at all.

After making changes to `/etc/default/grub`, make them take effect:

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/opensuse/grub.cfg
```

Note: if you just run `grub2-mkconfig` by itself, it writes to `/boot/grub2/grub.cfg`, which is not what's used when you're booting with EFI.

## Polkit rules for multi-session hibernate

By default, hibernate won't work without an interactive password prompt if the system detects more than one user logged in. This includes having a root shell open via `sudo -i`, which means it will trip you up more often than you'd expect.

Create `/etc/polkit-1/rules.d/10-local-privs.rules`:

```javascript
polkit.addRule(function(action, subject) {
       if (
               action.id == "org.freedesktop.login1.hibernate-multiple-sessions"
               ||
               action.id == "org.freedesktop.login1.suspend-multiple-sessions"
       ) {
               return polkit.Result.YES;
       }
});
```

## KDE Plasma power settings

Go to System Settings > Power Management and set the following for both "On AC Power" and "On Battery":

- **When laptop lid closed**: Sleep
- **When sleeping, enter**: Standby, then hibernate

## Sleep-then-hibernate timing

Create `/etc/systemd/sleep.conf.d/sleep-then-hibernate.conf`:

```ini
[Sleep]
HibernateDelaySec=9000
```

This is the piece that ties it all together. When I close the laptop lid, it goes into D3 sleep for 9000 seconds (two and a half hours). If I don't wake it by opening the lid, it transitions to full hibernation. I get the convenience of a quick resume when I'm just moving between rooms or going to dinner, and the power savings of hibernation when the laptop sits overnight or over the weekend.

Reboot, and you should be in business.

```bash
sudo reboot
```
