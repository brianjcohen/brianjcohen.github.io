---
title: You should probably use openSUSE-repos
description: 
date: 2025-05-01T15:23:43+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

OpenSUSE occupies this odd space in the world of Linux distributions, where it's not *quite* as polished as, say, Fedora or Ubuntu, but also not nearly as rough-around-the-edges as, say, Arch.  

A great example of something that should probably be a certain way immediately after installation, but isn't, is the curious matter of [openSUSE-repos](https://github.com/openSUSE/openSUSE-repos).

I've spent exactly zero time reading its source code, but according to its README.md, the general idea seems to be that instead of having your OpenSUSE core repositories statically configured in `/etc/zypp/repos.d`, this service will supply the up-to-date and correct list of repositories to `zypper`, periodically refreshing itself against a remote server via something called [Repository Index Service (RIS)](https://en.opensuse.org/openSUSE:Standards_Repository_Index_Service). 

I first noticed this existed when I came across a [Reddit thread](https://www.reddit.com/r/openSUSE/comments/1j24zs1/after_9_years_zyppers_parallel_downloading/) about how `zypper` was now capable of parallel downloading (yay!).  Many comments on the thread were pointing out that you could easily enable it by setting the environment variable `ZYPP_PCK_PRELOAD=1` and `ZYPP_CURL2=1` .  Some people went so far as to create aliases to `zypper` that included these variables.  

Then on the OpenSUSE Forums I found this:
https://forums.opensuse.org/t/question-correct-usage-of-new-zypper-parallel-download-in-tumbleweed/184074/4

> Users of openSUSE-repos on Tumblweed gained `mediahandler=curl2` as part of the [repository urls](https://github.com/openSUSE/openSUSE-repos/blob/main/opensuse-tumbleweed-repoindex.xml) as well as preset `ZYPP_PCK_PRELOAD=1 via /etc/profile.d/opensuse_repos.sh` with the latest openSUSE-repos update.

Curiously, `ZYPP_CURL2=1` appears to not really be a thing. So much for Reddit. It's a good thing we haven't trained all the world's LLM's on Reddit conversations. Right? Right?

So I took the leap (no pun intended).

## Install it

`sudo zypper install openSUSE-repos-Tumbleweed`

By the way, there are corresponding packages for Leap, MicroOS, Slowroll, as well as -NVIDIA packages for all of them if you happen to be using an NVIDIA graphics card and need those drivers.

This will setup the service, disable your old default OpenSUSE repositories, and install the `/etc/profile.d` entry that enables parallel downloading. That's all you gotta do. 

## References

https://github.com/openSUSE/openSUSE-repos
https://www.reddit.com/r/openSUSE/comments/1j24zs1/after_9_years_zyppers_parallel_downloading/
https://forums.opensuse.org/t/question-correct-usage-of-new-zypper-parallel-download-in-tumbleweed/184074/4
https://forums.opensuse.org/t/question-correct-usage-of-new-zypper-parallel-download-in-tumbleweed/184074/10