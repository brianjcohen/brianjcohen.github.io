---
title: An AI-First System Runbook
description: Claude audited my Tumbleweed machine, wrote the runbook and the backup script, and keeps both maintained. If the disk dies, Claude is also the one that's supposed to read the runbook and rebuild the machine.
date: 2026-08-16T12:00:00-04:00
draft: true
tags:
  - linux
  - ai
showTags: true
---
## Background

I've been running OpenSUSE Tumbleweed on a Framework laptop for a few years. A lot of the posts on this blog came out of fights with it: sleep and hibernate, ssh-agent under Wayland, the CrowdStrike sensor, printers.

After each fight I'd leave myself a note in Obsidian. "Sleep, Hibernate, Grub." "ssh-agent, ksshaskpass." "Restoring from a backup." I ended up with a dozen of these, and they rot. Some of mine, I found out later, were just wrong.

Meanwhile the machine itself is the sum of hundreds of small changes. A kernel flag for hibernate. A polkit rule so suspend stops asking for a password. A udev rule so unplugging the Thunderbolt dock doesn't wake the laptop out of hibernate while it's zipped in a bag. (That last one really happened. I found the laptop hot, at the LUKS prompt, in my backpack.) None of this is written down in one place, so if the SSD died, rebuilding would mean digging through stale notes and hoping I remember the rest.

If you complain about this online you get told to use NixOS, or an immutable distro like Silverblue or MicroOS. Those really do solve the problem. But everything on NixOS has to go through Nix, and my machine is weird in the wrong ways for that: the CrowdStrike sensor is an unsigned vendor RPM that still wants OpenSSL 1.1, my codecs come from packman, my Brother scanner driver is ancient, and there's a Windows guest under KVM that won't boot without its vTPM state. Immutable distros have the same problem from the other direction. The read-only base is great until you need to install something it didn't anticipate.

So I did something else. I kept my normal, mutable Tumbleweed install, and I handed the documentation problem to Claude.

## The setup

There's one document at the center of this, the System Runbook. The first line of it:

> **Audience:** an agent (human or AI) running as root on this machine, tasked with either repairing the existing system or rebuilding it from scratch.

The runbook is not written for me. It's written for Claude, and Claude is involved at every stage:

1. **Claude wrote it.** I pointed Claude Code at my pile of old notes and at the live machine, and had it reconcile the two into one document. This caught real errors: a config filename I'd recorded wrong, a hibernate delay in my notes that didn't match the file on disk, and several hand-made systemd timers I'd never documented at all. One of those timers runs an unattended `zypper dup` every week. I set that up myself at some point and had completely forgotten it existed.
2. **Claude wrote the backup script.** More below.
3. **Claude maintains it.** I have a `/runbook-sync` skill that scans `/etc`, `/var`, `/root`, my home directory, installed packages, enabled services, and backup health, compares all of it against the runbook, and appends corrections to a changelog. The rest of the document always describes desired state; the changelog is the only place history goes.
4. **Claude is supposed to use it.** The runbook defines two modes. Repair mode: run the Verify commands in each section, fix only what diverges. Rebuild mode: work through the phases in order, from bare OS install to a final checklist. My disaster recovery plan is to install Tumbleweed on the new disk, give Claude root, and hand it this document and the backup drive.

## What the runbook looks like

Since the reader is an agent, every section states what should be true and then how to prove it. Verify blocks everywhere:

```
firewall-cmd --get-active-zones     # trusted should be on tailscale0
grep TIMELINE_CREATE /etc/snapper/configs/root    # should say "no"
ffmpeg -decoders | grep hevc        # if empty, iPhone videos are broken
```

Full file contents are inlined for everything hand-made: the custom systemd units, the polkit rule, the udev rules. Most of the system can be reconstructed from the document alone, without touching a backup. The exceptions are secrets and user data, and the runbook says where those live instead.

The document also records *why* things are the way they are, because an agent without that context will "fix" them. Example: `security=apparmor` is on my kernel command line ([originally for snapd](https://brianjcohen.github.io/posts/snap-on-opensuse-tumbleweed/), back when Tumbleweed moved to SELinux by default). But `/etc/selinux` still contains an enforcing config and an `.autorelabel` flag file. All of that is inert today. If the kernel flag ever got lost, though, the machine would boot into enforcing SELinux and relabel the entire filesystem. The runbook spells this out so that nobody, human or AI, "cleans up" the leftovers into a booby trap.

And some entries exist to say what must NOT be present. There is deliberately no `ssh` service in the firewalld public zone; this machine is reachable over Tailscale only, and a failed LAN ssh is not a bug to fix by opening the port. There is no `nvme.noacpi=1` kernel flag anymore; that was an 11th-gen Framework fix I'd copied onto a 13th-gen board where it does nothing. I know these entries are necessary because I'm the one who added both of those things in the first place.

## The backup script

The audit also showed me that my backups didn't cover what I assumed they covered.

I'd been assuming snapper had my back. Mostly it doesn't. On my machine snapper's only config is the root filesystem, `/var` is its own btrfs subvolume, and timeline snapshots are off. So snapper protects `/` across package operations, and that's it. And snapshots die with the disk anyway.

The actual protection is now two restic repos on a WD Elements USB drive:

1. **Home.** Deja Dup (the Flatpak, which uses restic under the hood) backs up `$HOME`.
2. **Everything root-owned.** Deja Dup runs as my user, so it can't read `/root` or the rest of the root-owned tree, and I wasn't going to loosen those permissions for a backup tool. Instead a root-owned systemd timer runs a script Claude wrote, covering `/root`, `/etc`, `/usr/local`, Docker's named volumes, and a hand-picked list of paths under `/var`.

The `/var` list took some thought. `/var` on this machine is about 228 GB, and almost all of it can be regenerated: 145 GB of Docker layers that rebuild from Dockerfiles, 44 GB of VM disk images whose guests reinstall from ISOs, flatpaks, logs. The irreplaceable part is small and specific:

* `/var/lib/bluetooth` -- link keys, so the Magic Keyboard doesn't need re-pairing
* `/var/lib/tailscale` -- node identity, so the machine keeps its spot in my tailnet
* libvirt's nvram and vTPM state -- the Windows guest will not boot without these, even with its disk image intact
* crontabs, and the Docker volumes with real data in them

The whole backup is about 2 GB instead of 230. Every include and exclude has a one-line comment explaining it, so the next agent that finds a new directory under `/var/lib` can decide whether it belongs on the list.

One gotcha from this that's worth passing along even if you ignore everything else in this post: the drive normally sits unmounted, and a backup timer that fires with no drive attached exits 0 without backing up anything. `systemctl status` shows a healthy timer either way. A skipped run and a successful run look identical. The fix was a udev rule that starts the backup when the drive is plugged in, plus a rule in the runbook: verify backups by listing the restic snapshots, never by looking at the unit.

The restic password is in Bitwarden. After a disk failure, the copy on the machine is gone along with everything else, and restic has no password recovery. Getting it out of Bitwarden is step one of rebuild mode.

## Doesn't documentation drift?

Yes. That's the standard objection to any runbook, and it's why NixOS people are smug (deservedly).

But the reason documentation drifts is that nobody re-checks it. I never once audited my old Obsidian notes against the machine, which is why some of them were wrong for years. Comparing an entire `/etc` against a prose document is tedious, thorough work that no human sits down and does. Claude will happily do it, and now it's a slash command I run after making changes. The machine changes, the sync runs, the runbook gets corrected, the changelog records it.

## The tradeoff

NixOS rebuilds with one command. My rebuild is Claude working through phases for an afternoon. For one laptop that gets rebuilt maybe every few years, fine. In exchange I keep a completely normal Tumbleweed system where vendor RPMs install and ancient scanner drivers work, and nothing needs to be packaged or containerized just to exist.

And there's one more thing I like about this arrangement: the runbook keeps getting better on its own. Every time the sync runs it drags something undocumented into the light -- a timer I forgot, a config that drifted, a `cloudflared` unit with a live tunnel token still sitting in it that had no business being there. That last one is currently sitting in the changelog under "open, pending decision," which is more than my old notes ever did for me.
