---
title: The stale checksum that taught me zypper wasn't refreshing my repos
description: A digest verification error led me to discover that zypper's autorefresh is off by default, my mental model of zypper dup was wrong, and several of my repos had been quietly rotting for months.
date: 2026-06-02T09:30:00-04:00
draft: false
tags:
  - linux
showTags: true
---
## Background

A routine `zypper dup` threw this at me:

```
Warning: Digest verification failed for file 'netmaker-desktop-1.5.1-2.x86_64.rpm'

 expected e7fbb4b31d2f5e3e563419024a84fbec79e2525b73317a46fa4c539c227d4d53
 but got  bfe7323ffd41e7b038b4faae99a20ad79d024a1815b4909e79a16a077fa0d6d7
```

A checksum mismatch, with the usual offer to type the first four characters of the *wrong* hash to override the check. I emailed the vendor's support instead. They replied, reasonably:

> I checked the checksum in the metadata on the rpm server and it matches the rpm file. A colleague installed v1.5.1 fine. It's possible the metadata is stale — I recommend refreshing the repository.

And I got cocky. *Refresh the repository?* `zypper dup` refreshes before every run, everybody knows that. I was halfway through the smug reply before I decided to check my own machine first. That decision is the only reason this post is about zypper and not about me.

## The thing I was wrong about

Two facts fell out immediately. The cached metadata for that repo was dated April 7th. It was June. And:

```bash
zypper lr -d netmaker-desktop-repo
# Autorefresh : Off
```

My belief that "`zypper dup` refreshes everything first" is only true for repos with **autorefresh enabled**. Repos that opt out get refreshed *only* by an explicit `zypper ref`. This one had never opted in, so `dup` had been skipping it for two months.

And the checksum the error called "expected" was sitting right there in the stale metadata:

```bash
zcat /var/cache/zypp/raw/netmaker-desktop-repo/repodata/*-primary.xml.gz | grep -A2 'rel="2"'
# <version epoch="0" ver="1.5.1" rel="2"/>
# <checksum type="sha256">e7fbb4b31d2f...</checksum>
```

Not an authoritative truth from the server — a two-month-old memory my own machine was clinging to. The vendor had rebuilt `1.5.1-2` in place, same version, new bytes. Support was right. I was the stale one.

## How much was this costing me?

The checksum error wasn't the disease, just the one symptom loud enough to fail. So I refreshed everything and ran `zypper lu`. VS Code was eight releases behind:

```
v | vscode         | code             | 1.114.0 | 1.122.1
v | hardware_razer | openrazer-daemon | 3.12.1  | 3.12.3
```

`zypper refresh` doesn't download packages; it syncs the **catalog** — the metadata listing what exists and at what version. Everything `dup` knows, it knows from that catalog. So a stale catalog gives you two failure modes from one cause:

- **Silent drift.** A new version ships, never enters your catalog, never gets offered. That's vscode: no error, just old software for months.
- **A checksum that stops matching.** A package already in your catalog gets rebuilt in place. That's netmaker: the loud one, and the only reason I noticed.

If that vendor had bumped their release number instead of rebuilding in place, I'd have gotten the silent failure too and never known. The one mercy: the openSUSE repos ship with autorefresh on, so the OS and security updates were never at risk. The rot was confined to the third-party repos that opted out.

## You can't fix it with a default

My next thought was to make autorefresh the default. There's no knob — it's baked into libzypp, and `zypper ar` defaults `--refresh` to false. A default wouldn't help anyway: the painful repos (vscode, tailscale, the Google ones) had `autorefresh=0` written explicitly by vendor install scripts copy-pasting a Fedora/RHEL template, where the field is meaningless. And a vendor update can rewrite its own `.repo` and reset the flag whenever it likes. You want freshness to be a property of the system, not a negotiation with twenty vendors.

## The remedy

The fact I'd never internalized: a manual `zypper ref` refreshes **all** enabled repos regardless of the flag. So I don't need every repo to autorefresh — I need something to run `zypper ref` on a schedule. That's exactly what Fedora's `dnf-makecache.timer` does. Two files:

```ini
# /etc/systemd/system/zypper-refresh.service
[Unit]
Description=Refresh all zypper repository metadata
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/zypper --non-interactive refresh
```

```ini
# /etc/systemd/system/zypper-refresh.timer
[Timer]
OnBootSec=10min
OnUnitActiveSec=6h
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now zypper-refresh.timer
sudo zypper mr -a -r   # flip the existing flags on too, while we're at it
```

Now everything is at most six hours stale no matter what any vendor wrote.

## The tradeoff

Turning on refresh-everything is an honesty pass on your repo list: the moment zypper actually tried them all, the skeletons came out. Chiefly `antigravity`, a 737 MB experiment I'd installed god-knows-when with `gpgcheck=0` and completely forgotten. I removed it and felt lighter, in two senses.

That's the deal. The old way was serene and uninformed; the new way works and hands you a punch-list of everything you'd been ignoring. I'll take aware-and-briefly-busy over blissful.

The system was working exactly as documented. I just hadn't read the documentation, and nearly emailed a support tech to tell *him* he was wrong about it. Refresh your repos — and read the man page before you draft the smug reply.

## References

- `man zypper` — `refresh`, `addrepo`, and the `autorefresh` flag
- Fedora's `dnf-makecache.timer`, the prior art I was reinventing
