---
title: Using ssh-agent and ssh-askpass under Wayland on OpenSUSE
description: Setting up ssh-agent with graphical passphrase prompts under Wayland on OpenSUSE Tumbleweed with KDE Plasma and systemd.
date: 2025-04-03T09:54:33+00:00
draft: false
tags:
  - linux
showTags: true
---
*Updated August 2026: added the missing `environment.d` step, corrected my explanation of which part of the unit is actually doing the work, and fixed a bug in the `systemd` unit. Re-verified against openSUSE Tumbleweed 20260814, openssh 10.5p1, and Plasma 6.7.4.*

## Background

As of early 2025, out of the box, I found that the steps in this article were necessary in order to get OpenSUSE Tumbleweed running `ssh-agent` at boot, my keys loaded at login, and a graphical `ssh-askpass` configured to fire when user interaction is required (such as for keys that contain passphrases, or SK keys tied to a physical security device such as Yubikey). 

These instructions presume a KDE Plasma desktop although most of this is transferable to other desktop environments with a few tweaks.  I'm also guessing that some of this is helpful to users of other Linux distributions that use `systemd`. 

The first problem on Tumbleweed is that it doesn't even ship with a `systemd` unit service to start `ssh-agent`.  Looking in `/usr/lib/systemd/user` I can see `gpg-agent-ssh.socket` exists along with `gpg-agent.service` (although both are disabled by default) which offers some evidence that the "OpenSUSE way" is to use `gpg-agent` and its SSH support instead. I originally couldn't find any documentation supporting that theory, and having since gone looking, I can now say it's a dead end for my use case: `gpg-agent`'s SSH emulation has no FIDO/SK support at all, so it simply cannot hold an `ecdsa-sk` or `ed25519-sk` key. So, we're going to create a `systemd` unit file that starts `ssh-agent`. 

The second problem is that, under Wayland, the `WAYLAND_DISPLAY` environment variable doesn't get set until after the desktop environment has run its startup scripts (for KDE Plasma these are the scripts in `~/.config/plasma-workspace/env`).  If you Google around, you'll find [blog posts](https://dev.to/manekenpix/kde-plasma-ssh-keys-111e) and Reddit comments telling you to start `ssh-agent` in `~/.config/plasma-workspace/env` entries and to set the `SSH_ASKPASS` environment variable and to run `ssh-add` from Plasma's "Autostart".   This will only work in X11.

Under Wayland, `ssh(1)` and `ssh-add(1)` and `ssh-keygen(1)` will skip firing `ssh-askpass` due to the lack of a `WAYLAND_DISPLAY` value. The relevant code is in `read_passphrase()` in [`readpass.c`](https://github.com/openssh/openssh-portable/blob/master/readpass.c):

```c
if (((s = getenv("DISPLAY")) != NULL && *s != '\0') ||
    ((s = getenv("WAYLAND_DISPLAY")) != NULL && *s != '\0'))
        allow_askpass = 1;
if ((s = getenv(SSH_ASKPASS_REQUIRE_ENV)) != NULL) {
        if (strcasecmp(s, "force") == 0) {
                use_askpass = 1;
                allow_askpass = 1;
        } else if (strcasecmp(s, "prefer") == 0)
                use_askpass = allow_askpass;
        else if (strcasecmp(s, "never") == 0)
                allow_askpass = 0;
}
...
if ((flags & RP_USE_ASKPASS) && !allow_askpass)
        return (flags & RP_ALLOW_EOF) ? NULL : xstrdup("");
```

That last branch is the one that really hurts if you use SK keys. `ssh-agent` itself calls `read_passphrase()` with `RP_USE_ASKPASS` when it needs your PIN at signing time. With no display variable in scope, it doesn't error — it silently hands back an empty string, and your authentication just fails for no visible reason. This is why the *agent's own* environment has to carry `WAYLAND_DISPLAY`, not merely your shell's.

The problem was identified all the way back in 2015, in this bug report:

https://kde-bugs-dist.kde.narkive.com/tHqRlvxr/plasmashell-bug-380311-new-no-way-to-launch-ssh-agent-with-interactivity-under-wayland

The bug reporter, along with David and Nate from the KDE team identified that you could work around this by mule-ing the variables around. Or, you could wait until the startup of Plasma components such as Kwin were moved to `systemd`, which eventually happened in 2018. David writes:

> putting ssh-agent between the two and being sure to run  
> dbus-update-activation-env --systemd at the end of the script would then work.

That advice was right for the Plasma of its day. As I'll explain below, modern Plasma does the environment import for you, so the surviving trick is purely one of *ordering*.

## Setup

Okay let's wire this up. There are four pieces. Replace `myuser` with your own username throughout.

### 1. Set the askpass variables early

I'm going to use `/etc/profile.d` because SDDM runs your session through a login shell, so anything here lands in the environment Plasma inherits.

Create `/etc/profile.d/sshaskpass.sh`:
```bash
#!/bin/bash
export SSH_ASKPASS_REQUIRE=prefer
export SSH_ASKPASS=/usr/libexec/ssh/ksshaskpass
```
* `SSH_ASKPASS_REQUIRE` is set to `prefer`, which will be an instruction to `ssh` to make use of a graphical askpass utility (if it can) rather than prompting you in the terminal. 
* `SSH_ASKPASS` is set to the full path of our favorite compatible askpass implementation.

On current Tumbleweed the package you want is `ksshaskpass6`, not `ksshaskpass` — `zypper in ksshaskpass6`. It installs `/usr/bin/ksshaskpass` and symlinks it into `/usr/libexec/ssh/`.

A question I get asked: why not `SSH_ASKPASS_REQUIRE=force`, which per the code above sets `allow_askpass = 1` unconditionally and would seem to make the whole `WAYLAND_DISPLAY` problem go away? Two reasons. First, it doesn't actually work — OpenSSH would happily exec `ksshaskpass`, but `ksshaskpass` is a Qt application and still can't open a window without `WAYLAND_DISPLAY`. You'd trade a silent empty passphrase for a Qt platform-plugin error, which is at least louder but no more functional. Second, `force` means you get a GUI prompt even when you've SSH'd *into* this machine and genuinely want a terminal prompt. `prefer` is the right setting.

### 2. Tell everything where the agent socket will be

This is the step I originally left out of this article, and it's the one that makes the difference between "the agent is running" and "the agent is running and anything can find it."

Create `~/.config/environment.d/ssh-agent.conf`:
```
SSH_AUTH_SOCK=$XDG_RUNTIME_DIR/ssh-agent.socket
```

`systemd` reads `environment.d` when it starts your user manager, before any user unit runs, so `SSH_AUTH_SOCK` ends up in the environment of the user manager and therefore of plasmashell, your autostart script, your terminals, and every app you launch. Without this file you get a perfectly healthy agent listening on a socket that nothing knows the path to.

### 3. Create the systemd user service

In `~/.config/systemd/user/ssh-agent.service`:

```ini
[Unit]
Description=SSH Key Agent
# Load-bearing: kwin publishes WAYLAND_DISPLAY into the systemd user manager
# environment once the compositor socket exists. Starting After= it is what lets
# ssh-agent inherit WAYLAND_DISPLAY.
After=plasma-kwin_wayland.service
# So the socket exists before plasmashell runs the autostart entry below.
Before=plasma-plasmashell.service
PartOf=graphical-session.target

[Service]
Type=simple
# Clear any socket left behind by an unclean logout.
ExecStartPre=/bin/rm -f %t/ssh-agent.socket

Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
ExecStart=/usr/bin/ssh-agent -a ${SSH_AUTH_SOCK} -D
ExecStopPost=/bin/rm -f %t/ssh-agent.socket

# ssh-agent's SIGTERM handler calls _exit(2), so 2 is the normal shutdown path.
# Without this the unit lands in 'failed' state at every logout.
SuccessExitStatus=2

# Restart if unexpectedly dies
Restart=on-failure
RestartSec=5

[Install]
WantedBy=graphical-session.target
```

And enable it:

`systemctl enable --user ssh-agent.service`

Notice a few things:
* `After=` and `Before=` are ensuring that we run this service "between" Kwin and plasmashell, but within the same target as the two of them. **This ordering is the entire trick.** Plasma's `startplasma` calls `syncDBusEnvironment()` and `importSystemdEnvrionment()` before it starts `plasma-workspace.target`, which pushes your whole login-shell environment (including `SSH_ASKPASS` from step 1) into the `systemd` user manager and D-Bus. Kwin then publishes `WAYLAND_DISPLAY` the same way once the compositor socket exists. So by the time a unit ordered `After=plasma-kwin_wayland.service` starts, everything it needs is already in the manager environment and it simply inherits it.
* Earlier versions of this article had an `ExecStartPre=/usr/bin/dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY ...` line here, following the 2015 advice, and credited it as the mechanism. That was wrong, and I've removed it. `dbus-update-activation-environment` can only propagate variables that are already in *its own* environment, which it inherits from the same user manager — so at this point in startup it can only re-push what's already there. It's a no-op. Harmless, but it isn't what makes this work, and leaving it in obscures the fact that the ordering is.
* No `ExecStop`. I used to have `ExecStop=/usr/bin/ssh-agent -k` here, which cannot work: `ssh-agent -k` kills the agent named by `$SSH_AGENT_PID`, and because we run the agent in the foreground with `-D`, that variable is never set. It failed with `SSH_AGENT_PID not set, cannot kill agent` at every single logout and left the unit in `failed` state. `systemd` sends `SIGTERM` to the agent anyway, which is what was actually clearing my keys all along, so the fix is just to delete the line and let it do its job.
* When the service stops, `systemd` kills the agent. This ensures that your keys aren't being held in a running agent after you logout. 

### 4. Load your keys at login

Next, we configure Plasma to load our keys into the agent at login. You can store your autostart script wherever you want, and name it whatever you want.

In `~/.config/autostart/ssh-add.sh.desktop`:
```ini
[Desktop Entry]
Exec=/home/myuser/store/script/autostart-ssh-add.sh
Icon=dialog-scripts
Name=ssh-add.sh
Type=Application
X-KDE-AutostartScript=true
```

In `/home/myuser/store/script/autostart-ssh-add.sh`:

```bash
#!/bin/bash
ssh-add -q ~/.ssh/id_ecdsa_sk
ssh-add -q ~/.ssh/id_ed25519_sk
```

And mark as executable.  

`chmod u+x /home/myuser/store/script/autostart-ssh-add.sh`

Here I have it explicitly loading two of my keys, both of which are -SK keys tied to my Yubikey. You can put whatever keys you want here. 

Finally, reboot.  

`sudo reboot`

To confirm it worked: open a terminal and run `ssh-add -l`, which should list your keys, and check `systemctl --user status ssh-agent.service` for a clean `active (running)` with no failures in the log.

## Final Thoughts

This setup is sufficient for me because I use SK keys tied to my Yubikey, but I do not use regular keys that have passphrases.  For that situation, you may want to use Kwallet to store your passphrases, allowing your keys stored within your agent to be unlocked at login and used throughout your session.  Tumbleweed ships `plasma-kwallet-pam.service`, which unlocks Kwallet from your PAM credentials at login, and `ksshaskpass` will store into and read from Kwallet automatically once it's set up. While I've never done it myself, it's my understanding that you just need to launch Kwallet and enable it. 

Since I first wrote this, two other options have turned up that are worth knowing about, neither of which displaced the above for me:

* **`gcr-ssh-agent`.** Tumbleweed now ships `gcr-ssh-agent.service` and `gcr-ssh-agent.socket` (from the `gcr-ssh-agent` package), both disabled by default. It's **socket-activated**, which is genuinely elegant — socket activation sidesteps the whole ordering problem, because the socket exists from the moment your user manager starts and the agent is spawned on first connection. If you're using ordinary keys it's worth a look. It didn't work for me: it's GNOME's agent, I see no FIDO/SK support in it, and there are [reports of it hanging SSH connections on openSUSE](https://forums.opensuse.org/t/gcr-ssh-agent-causes-ssh-hang-with-ed25519-key/189481).
* **`gpg-agent`.** As noted above, no FIDO/SK support. Fine if you're already living in a GnuPG world and don't use security keys.

As of Plasma 6.7.4 and openssh 10.5p1, neither KDE nor the `openssh` package ships an `ssh-agent` user unit, so a hand-rolled unit is still the answer. `openssh` still ships no unit at all, and of the thirty-odd `plasma-*.service` units in `/usr/lib/systemd/user`, none of them is an ssh-agent — KDE's contribution here is `ksshaskpass` and `plasma-kwallet-pam.service`, and that's it.

The help I received from my good friend, and very talented engineer, [Drew Vogel](https://www.linkedin.com/in/drewpvogel/) was invaluable as I was working through this problem.  
