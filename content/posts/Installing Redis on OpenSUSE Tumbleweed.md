---
title: Installing Redis on OpenSUSE Tumbleweed
description: How to install Redis on OpenSUSE Tumbleweed, including fixing the permissions issues that the package doesn't handle out of the box.
date: 2026-02-18T10:00:00-05:00
draft: false
tags:
  - linux
showTags: true
---
## Background

Redis is in the Tumbleweed repositories, which is great. What's less great is that the package doesn't set up its own directory permissions correctly, so the service will fail to start right after install. If you're a reader of my posts you'll know what comes next.

## Install it

```bash
sudo zypper install redis
```

## Fix permissions

The package creates a `redis` user and group but doesn't properly set ownership on the directories that Redis needs to write to. Without this step, the service will fail on startup with permission errors:

```bash
sudo mkdir -p /var/log/redis /var/lib/redis/default /run/redis
sudo chown redis:redis /var/log/redis /var/lib/redis/default /run/redis
```

If the log file already exists:

```bash
sudo chown redis:redis /var/log/redis/default.log
```

## Enable and start the service

The Redis package on Tumbleweed uses a systemd template unit, which means you reference a named instance. The default instance is called `default`:

```bash
sudo systemctl enable --now redis@default
```

## Verify it's running

```bash
redis-cli ping
```

You should get `PONG`.

## Configuration

The config is split across two files:

- `/etc/redis/default.conf` — instance-specific settings (port, log path, data directory, pid file). This is where your customizations go.
- `/etc/redis/includes/redis.defaults.conf` — all the standard Redis defaults. Don't edit this one; it'll get overwritten on package updates.

The template unit approach means you could theoretically run multiple Redis instances by creating additional conf files and enabling `redis@whatever`. For most people, `default` is all you need.
