---
title: Using ssh-agent and ssh-askpass under Wayland on OpenSUSE
description: 
date: 2025-04-03T09:54:33+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

As of early 2025, out of the box, I found that the steps in this article were necessary in order to get OpenSUSE Tumbleweed running `ssh-agent` at boot, my keys loaded at login, and a graphical `ssh-askpass` configured to fire when user interaction is required (such as for keys that contain passphrases, or SK keys tied to a physical security device such as Yubikey). 

These instructions presume a KDE Plasma desktop although most of this is transferable to other desktop environments with a few tweaks.  I'm also guessing that some of this is helpful to users of other Linux distributions that use `systemd`. 

The first problem on Tumbleweed is that it doesn't even ship with a `systemd` unit service to start `ssh-agent`.  Looking in `/usr/lib/systemd/user` I can see `gpg-agent-ssh.service` exists along with `gpg-agent.servce` (although both are disabled by default) which offers some evidence that the "OpenSUSE way" is to use `gpg-agent` and its SSH support instead. Unfortunately I can't find any documentation anywhere supporting this theory.  So, we're going to create a `systemd` unit file that starts `ssh-agent`. 

The second problem is that, under Wayland, the `WAYLAND_DISPLAY` environment variable doesn't get set until after the desktop environment has run its startup scripts (for KDE Plasma these are the scripts in `~/.config/plasma-workspace/env`).  If you Google around, you'll find [blog posts](https://dev.to/manekenpix/kde-plasma-ssh-keys-111e) and Reddit comments telling you to start `ssh-agent` in `~/.config/plasma-workspace/env` entries and to set the `SSH_ASKPASS` environment variable and to run `ssh-add` from Plasma's "Autostart".   This will only work in X11.  Under Wayland, `ssh(1)` and `ssh-add(1)` and `ssh-keygen(1)` will [skip firing](https://github.com/openssh/openssh-portable/blob/master/readpass.c#L265) `ssh-askpass` due to the lack of a `WAYLAND_DISPLAY` value.   The problem was identified all the way back in 2015, in this bug report:

https://kde-bugs-dist.kde.narkive.com/tHqRlvxr/plasmashell-bug-380311-new-no-way-to-launch-ssh-agent-with-interactivity-under-wayland

The bug reporter, along with David and Nate from the KDE team identified that you could work around this by mule-ing the variables around. Or, you could wait until the startup of Plasma components such as Kwin were moved to `systemd`, which eventually happened in 2018. David writes:

> putting ssh-agent between the two and being sure to run  
> dbus-update-activation-env --systemd at the end of the script would then work.

## Setup

Okay let's wire this up.  First, we want to set some environment variables EARLY. I'm going to use `/etc/profile.d` although my sense is that we could also use `environment.d`. 

Create `/etc/profile.d/sshaskpass.sh`:
```
#!/bin/bash
export QT_QPA_PLATFORM="wayland"  
export SSH_ASKPASS_REQUIRE=prefer
export SSH_ASKPASS=/usr/libexec/ssh/ksshaskpass
```
* `SSH_ASKPASS_REQUIRE` is set to `prefer`, which will be an instruction to `ssh` to make use of a graphical askpass utility (if it can) rather than prompting you in the terminal. 
* `SSH_ASKPASS` is set to the full path of our favorite compatible askpass implementation.
* `QT_QPA_PLATFORM` is set to `wayland` because... well actually I'm not really sure why, I copied this from somewhere.  

Next, we create a `systemd` unit service for my user:

In `~/.config/systemd/user/ssh-agent.service`:

```
[Unit]
Description=SSH Key Agent
# Explicitly place between kwin and plasmashell
After=plasma-kwin_wayland.service  
Before=plasma-plasmashell.service
PartOf=graphical-session.target  
  
[Service]
Type=simple
# Ensure clean environment and socket
ExecStartPre=/bin/rm -f %t/ssh-agent.socket
ExecStartPre=/usr/bin/dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY XDG_RUNTIME_DIR SSH_ASKPASS SSH_ASKPASS_REQUIRE
  
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket 
ExecStart=/usr/bin/ssh-agent -a ${SSH_AUTH_SOCK} -D
ExecStop=/usr/bin/ssh-agent -k
ExecStopPost=/bin/rm -f %t/ssh-agent.socket
  
# Restart if unexpectedly dies
Restart=on-failure  
RestartSec=5  
  
[Install]
WantedBy=graphical-session.target
```

And enable it:

`systemctl enable --user ssh-agent.service`

Notice a few things:
* `After=` and `Before=` are ensuring that we run this service "between" Kwin and plasmashell, but within the same target as the two of them. 
* When starting this service, we clear out any existing socket, and then we push a bunch of environment variables (including `WAYLAND_DISPLAY`) up using the `dbus-update-activation-environment` command recommended by Nate Graham, so that Plasmashell inherits them when it's time to run its startup and autostart scripts, and when you want to run `ssh` to make outbound connections. 
* When the service stops, it kills the agent. This ensures that your keys aren't being held in a running agent after you logout. 

Next, we configure Plasma to load our keys into the agent at login. Of course you should replace `brian` with your username, and of course you can store your autostart script wherever you want. You can also name it whatever you want. 

In `~/.config/autostart/ssh-add.desktop`:
```
[Desktop Entry]
Exec=/home/brian/store/script/autostart-ssh-add.sh  
Icon=dialog-scripts
Name=ssh-add.sh
Type=Application
X-KDE-AutostartScript=true
```

In `/home/brian/store/script/autostart-ssh-add.sh`:

```
#!/bin/bash
ssh-add -q ~/.ssh/id_ecdsa_sk
ssh-add -q ~/.ssh/id_ed25519_sk
```

And mark as executable.  

`chmod u+x /home/brian/store/script/autostart-ssh-add.sh`

Here I have it explicitly loading two of my keys, both of which are -SK keys tied to my Yubikey. You can put whatever keys you want here. 

Finally, reboot.  

`sudo reboot`

## Final Thoughts

This setup is sufficient for me because I use SK keys tied to my Yubikey, but I do not use regular keys that have passphrases.  For that situation, you may want to use Kwallet to store your passphrases, allowing your keys stored within your agent to be unlocked at login and used throughout your session.  While I've never done it myself, it's my understanding that you just need to launch Kwallet and enable it. 

The help I received from my good friend, and very talented engineer, [Drew Vogel](https://www.linkedin.com/in/drewpvogel/) was invaluable as I was working through this problem.  




