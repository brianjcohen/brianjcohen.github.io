---
title: Why I switched from Brave to Chrome
description: After years of using Brave Browser, I switched back to Chrome. The privacy tradeoffs weren't worth the constant friction of things not working.
date: 2026-02-10T12:00:00-05:00
draft: false
tags:
  - internet
showTags: true
---
## Background

I used Brave as my daily driver for several years. The pitch was compelling: a Chromium-based browser with built-in ad blocking and tracker prevention, no need for extensions, and a genuine commitment to user privacy. For someone who spends 100% of his workday at a computer (a lot of that in a browser), that sounded great.

And for a while, it was great. Brave Sync chains were genuinely excellent — syncing bookmarks, history, and open tabs across desktop and mobile without needing a Google account. The built-in Shields feature blocked ads and trackers out of the box. I felt good about de-Googling a significant part of my daily workflow.

But over time, I started to notice a pattern.

## Shields breaks things

Brave Shields is aggressive by default, and that's kind of the point. It blocks third-party scripts, cross-site cookies, fingerprinting, and a lot of remotely-loaded content. Most of the time you don't notice. But when something on a website doesn't work — a button that does nothing, a modal that won't load, a login flow that hangs — you're left wondering: is this site broken, or is it broken *for me*?

That question became a constant companion. Every time I hit a problem on a website, I'd have to go through the ritual: disable Shields for this site, reload, try again. If it works now, great, the site needed a third-party script that Shields was blocking. If it still doesn't work, re-enable Shields and go actually debug the problem. This tax on my attention added up, and over time I found myself reflexively disabling Shields on more and more sites, which kind of defeats the purpose.

## The passkey problem

The issue that finally pushed me over the edge was passkeys. I use Bitwarden as my password manager, and I've been migrating my most critical logins to passkeys stored in Bitwarden. This works flawlessly in Chrome — when a site requests passkey authentication, Bitwarden's browser extension intercepts it and presents its own UI for selecting the passkey.

In Brave, this was broken for AWS Console. Instead of Bitwarden's passkey UI, Brave would show the native Chromium passkey dialog, which only knows about hardware security keys and Google Account passkeys. My Bitwarden-stored passkeys were invisible. I couldn't log in to AWS.

I [raised this with Brave Community](https://community.brave.app/t/brave-bitwarden-passkeys-aws-console/648982). It works in Chrome. It works in Chromium. It doesn't work in Brave. This is exactly the kind of issue that erodes trust — a mission-critical login flow, silently broken by something the browser is doing differently, with no obvious indication of why.

## What I lost

Switching to Chrome meant giving some things up, and I want to be honest about the tradeoffs.

**Sync chains.** Brave's sync was great and didn't require a Google account. Chrome sync requires signing in to Google, which means Google knows your bookmarks, history, and open tabs. I've accepted this.

**Mobile ad blocking.** Brave on Android blocked ads and trackers natively. Chrome on Android doesn't support extensions, so there's no uBlock, no ad blocker, nothing. I'm seeing ads again on mobile and there's not much I can do about it.

**Mobile Bitwarden autofill.** On the topic of mobile, Bitwarden's autofill on Brave for Android was always flaky — sometimes it wouldn't trigger, sometimes it would fill the wrong field, sometimes it just wouldn't appear at all. On Chrome for Android it works reliably.

**Independence from Google.** When you sign into Chrome with a Google Workspace account, you get a degree of remote management from your Workspace admin. That's a double-edged sword — it means your organization can enforce policies on your browser, but it also means your browsing is more tightly integrated with Google's ecosystem. For a personal account this is less of a concern, but it's worth knowing about.

## What I gained

Things just work. That's it. That's the whole value proposition. Sites load correctly. Login flows complete. Passkeys work. I don't have to wonder if the problem is the website or my browser. I don't have to toggle Shields on and off. I don't have to file bug reports about basic authentication flows being broken.

On desktop, I run uBlock Origin Lite (the Manifest V3 version) which gives me most of the ad blocking and tracker prevention I had with Brave. It's not as aggressive as Shields, and that turns out to be a feature — it blocks the obvious stuff without breaking site functionality.

## The uncomfortable truth

The privacy-focused browser space is full of genuine, well-intentioned projects. Brave is one of them. But there's a fundamental tension between blocking the tracking and advertising infrastructure that most of the web is built on, and having that web work reliably. Every blocked script is a potential broken feature. Every fingerprinting countermeasure is a potential compatibility issue.

I don't like being tracked. But I spend my entire workday in a browser, and I just don't have time for things to not work. For me, the math eventually stopped adding up.
