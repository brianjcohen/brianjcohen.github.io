---
title: "ROLLBACK NOTES — ssh-agent cleanup 2026-08-16"
draft: true
---

# ssh-agent / ksshaskpass cleanup — 2026-08-16 — rollback notes

Changes made to `bcohen-framework` on 2026-08-16 while fact-checking the post
*"Using ssh-agent and ssh-askpass under Wayland on OpenSUSE"*. Everything here is
reversible. Read **"If it's broken after reboot"** at the bottom first if you're
in a hurry.

**Backups of every original file:** `~/.local/state/ssh-agent-cleanup-2026-08-16/`

```
etc-profile.d-sshaskpass.sh        <- /etc/profile.d/sshaskpass.sh
ssh-agent.service                  <- ~/.config/systemd/user/ssh-agent.service
plasma-env-ssh-agent-startup.sh    <- ~/.config/plasma-workspace/env/ssh-agent-startup.sh (deleted)
environment.d-ssh-agent.conf       <- ~/.config/environment.d/ssh-agent.conf (unchanged)
ssh-add.sh.desktop                 <- ~/.config/autostart/ssh-add.sh.desktop (unchanged)
autostart-ssh-add.sh               <- ~/store/script/autostart-ssh-add.sh (unchanged)
```

---

## What changed

### 1. `~/.config/systemd/user/ssh-agent.service` — three edits

**a. Removed `ExecStop=/usr/bin/ssh-agent -k`.** This never worked. `ssh-agent -k`
kills the agent named by `$SSH_AGENT_PID`, and because the unit runs the agent
foreground with `-D`, that variable is never set. Verified:

```
$ env -u SSH_AGENT_PID ssh-agent -k
SSH_AGENT_PID not set, cannot kill agent    # exit 1
```

The journal showed it failing at *every* logout going back to at least Jul 13:

```
Aug 03 20:18:05 ssh-agent[219327]: SSH_AGENT_PID not set, cannot kill agent
Aug 03 20:18:05 systemd[3361]: ssh-agent.service: Control process exited, code=exited, status=1/FAILURE
Aug 03 20:18:05 systemd[3361]: ssh-agent.service: Failed with result 'exit-code'.
```

Harmless in practice — systemd's SIGTERM was killing the agent all along
(`exiting on signal 15` in the same log), so keys were never left loaded after
logout. But the unit landed in `failed` state every time.

**b. Added `SuccessExitStatus=2`.** openssh's `ssh-agent` installs a SIGTERM
handler that calls `_exit(2)`, so exit code 2 *is* the normal shutdown path. Without
this, systemd reported `Main process exited, code=exited, status=2/INVALIDARGUMENT`
at every logout and (with `Restart=on-failure` set) could try to restart the agent
on the way out.

**c. Removed `ExecStartPre=/usr/bin/dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY XDG_RUNTIME_DIR SSH_ASKPASS SSH_ASKPASS_REQUIRE`.**

**This is the highest-risk change of the batch — revert this one first if anything
misbehaves.** Reasoning, and the evidence:

- `dbus-update-activation-environment` can only propagate variables that are already
  in its *own* environment, which it inherits from the systemd user manager. So it
  cannot introduce anything the manager doesn't already have.
- `systemd --user` is PAM-spawned and inherits almost nothing —
  `/proc/<pid>/environ` for it contains only `XDG_SESSION_TYPE=unspecified`.
- Everything else in `systemctl --user show-environment` therefore arrived via
  runtime `SetEnvironment` calls. Those come from `startplasma`, which calls
  `syncDBusEnvironment()` and `importSystemdEnvrionment()` **before**
  `startPlasmaSession()` (see `startkde/startplasma.cpp` in plasma-workspace),
  pushing the whole login-shell environment — including `SSH_ASKPASS` from
  `/etc/profile.d/sshaskpass.sh`, which SDDM's `wayland-session` sources via
  `$SHELL --login`. Kwin publishes `WAYLAND_DISPLAY` the same way.
- The unit is ordered `After=plasma-kwin_wayland.service`, so all five variables
  are already in the manager environment before `ExecStartPre` ever runs.

**Tested live, not just reasoned about.** Restarted the service with the line
removed; the new agent process still inherited everything:

```
$ tr '\0' '\n' < /proc/1416650/environ | grep -E 'WAYLAND|DISPLAY|ASKPASS'
XDG_RUNTIME_DIR=/run/user/1000
DISPLAY=:1
SSH_ASKPASS=/usr/libexec/ssh/ksshaskpass
SSH_ASKPASS_REQUIRE=prefer
WAYLAND_DISPLAY=wayland-0
```

The restart also produced a clean stop for the first time — no `SSH_AGENT_PID`
error, no `Failed with result 'exit-code'`.

The load-bearing pieces are `After=plasma-kwin_wayland.service` and
`/etc/profile.d/sshaskpass.sh`. **Neither was touched.**

### 2. `/etc/profile.d/sshaskpass.sh` — removed a dead comment

Deleted the commented-out `#export QT_QPA_PLATFORM="wayland"` line. The two live
`export` lines are byte-identical to before. Zero functional change.

### 3. Deleted `~/.config/plasma-workspace/env/ssh-agent-startup.sh`

Entire contents were:

```bash
#!/bin/bash
#[ -n "$SSH_AGENT_PID" ] || eval "$(ssh-agent -s)"
```

The only executable line was commented out — a leftover from the X11-era approach
the blog post argues against. It could not have been doing anything.
(`kwin-reorder.sh` in the same directory is unrelated and was left alone.)

### 4. Ran `systemctl --user daemon-reload` and restarted the service

All 5 keys were re-added afterward and `ssh-add -l` confirms them. Yubikey was
plugged in at the time. **Nothing about your keys, `~/.ssh`, or the autostart
script changed.**

---

## What was deliberately NOT changed

- `~/.config/environment.d/ssh-agent.conf` — this sets `SSH_AUTH_SOCK` and is what
  makes the whole thing discoverable. Untouched, and now documented in both the
  blog post and the runbook (it was missing from both).
- `~/.config/autostart/ssh-add.sh.desktop` and `~/store/script/autostart-ssh-add.sh`.
- `~/.ssh/` — nothing read, written, or moved. `id_ed25519_passphrase` remains the
  only passphrase-protected key and remains deliberately not autoloaded.
- `gcr-ssh-agent.socket` / `.service` — already `disabled`, left disabled.

---

## Documents updated

- `posts/ssh-agent and ssh-askpass under Wayland on OpenSUSE Tumbleweed.md` —
  added the missing `environment.d` step, corrected the causal explanation,
  fixed the unit, added `force` vs `prefer`, refreshed the alternatives section.
- `Personal Tech/OS and Software Setup Guides/System Runbook.md` — §4.4 and §11.1
  rewritten to match the new desired state; verify line in the final checklist updated.

---

## If it's broken after reboot

**Fastest full revert** (restores all three files to their 2026-08-16 state):

```bash
BK=~/.local/state/ssh-agent-cleanup-2026-08-16
cp -a "$BK/ssh-agent.service" ~/.config/systemd/user/ssh-agent.service
cp -a "$BK/plasma-env-ssh-agent-startup.sh" ~/.config/plasma-workspace/env/ssh-agent-startup.sh
sudo cp -a "$BK/etc-profile.d-sshaskpass.sh" /etc/profile.d/sshaskpass.sh
sudo chown root:root /etc/profile.d/sshaskpass.sh
systemctl --user daemon-reload
# log out and back in
```

**Targeted revert** — if the symptom is *"ksshaskpass never appears / SK key auth
silently fails"*, it's change 1c. Add this back under `[Service]` and re-login:

```ini
ExecStartPre=/usr/bin/dbus-update-activation-environment --systemd WAYLAND_DISPLAY DISPLAY XDG_RUNTIME_DIR SSH_ASKPASS SSH_ASKPASS_REQUIRE
```

**Diagnosing before reverting:**

```bash
systemctl --user status ssh-agent.service          # should be active (running), no failures
ssh-add -l                                          # should list 5 keys
systemctl --user show-environment | grep -E 'WAYLAND_DISPLAY|SSH_ASKPASS|SSH_AUTH_SOCK'
tr '\0' '\n' < /proc/$(pgrep -u brian -f 'ssh-agent -a')/environ | grep -E 'WAYLAND|ASKPASS'
```

If that last command doesn't show `WAYLAND_DISPLAY` and `SSH_ASKPASS`, change 1c is
the culprit. If it does show them and things still fail, the problem is elsewhere and
reverting won't help.

This file can be deleted once you've rebooted and confirmed everything works.
