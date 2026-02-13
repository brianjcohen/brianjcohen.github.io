---
title: What I learned going through old stored logins
description: Observations from auditing years of stored credentials — dead sites, broken logins, missing 2FA, and the scavenger hunt of changing your email address.
date: 2026-02-10T10:00:00-05:00
draft: false
tags:
  - internet
showTags: true
---
## Background

A couple years ago, I went through all of my stored logins as part of a password manager migration. Every single one.  I decided to visit each site, attempt to log in, and see where things stood.

I had accumulated hundreds of credentials over the years. What I found was, honestly, kind of fascinating in a depressing way.

## A lot of these sites are just... gone

This was the first surprise, although maybe it shouldn't have been. A non-trivial number of sites in my list simply don't exist anymore. The domain is parked, or it redirects to some holding page, or the DNS just doesn't resolve. Years of patronage, reduced to a 404.

Others have been acquired. Your account *might* still exist, folded into the acquiring company's database — and if you already had an account there under the same email, who knows which one survived. Or your data could have simply been deleted post-acquisition. There's no way to know without trying, and trying often doesn't tell you much either.

Then there are the ecommerce sites that changed platforms and apparently didn't migrate their user database, and the sites that did a unilateral purge at some point. Your account is just gone. No warning, no email, no ceremony.

## A lot of these passwords just don't work anymore

Even on sites that are still alive and kicking, there's a decent chance your stored password doesn't work. Sometimes it's because the site quietly forced a password expiration — either due to age or because they decided your password wasn't strong enough by whatever their current standards are. They may or may not have emailed you about it. If they did, it was probably years ago.

Other times, the login mechanism itself has changed. You signed up with a username, but now the site only accepts an email address. Or they've added mandatory SMS or email-based two-factor authentication since the last time you logged in, and now you have to go dig up your phone to get a code before you can even see your account settings.

On that note: the adoption of proper two-factor authentication on consumer-facing sites is surprisingly low. SMS and email codes are everywhere, and I encountered a handful of "magic link" setups, but almost nobody is offering TOTP, Duo, or FIDO2/U2F. The sites that handle your money and your medical records are using the same second factor as the site that sells you socks.

## Good luck deleting your account

A very small percentage of sites have any kind of account deletion or deactivation feature. It tends to be the SaaS services that offer this — the ones that actually have some regulatory pressure or competitive reason to let you leave cleanly. The rest? Your account just exists there, forever, with whatever data you gave it.

## The email change odyssey

I was also trying to update my email address on many of these accounts, which turned out to be its own special adventure.

Some services use your email as the login, some don't. The ones that don't tend to have a separate "email address" field buried somewhere in account settings, and finding it is a scavenger hunt every time. My Account? Account Settings? Profile? Contact Info? Personal Info? It's different on every single site.

You will be sent a lot of one-time verification codes. A *lot*. Sometimes to your old email, sometimes to your new email, sometimes to your phone, sometimes to all three.

Some sites simply won't let you change your email address at all. The email address is apparently some kind of primary key in their database, and the idea of updating it was not considered. If you've lost access to that address, your only option is to mothball the account and start fresh.

And here's a fun bonus: changing your email address on some services will re-subscribe you to marketing emails you had previously unsubscribed from under your old address. Logging back into a site after a long hiatus can do this too. Because apparently "unsubscribe" is stored per-email, not per-account, and re-engagement is more important than your preferences.

## Takeaways

None of this is earth-shattering, but going through it all in one sitting really drives home how fragile and inconsistent the state of online identity management is. We've built an entire digital economy on the assumption that people will maintain hundreds of accounts across hundreds of services, and then we've made it as difficult as possible to actually manage them.
