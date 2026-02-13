---
title: Running the Crowdstrike Falcon Sensor on OpenSUSE Tumbleweed
description: Installing Crowdstrike Falcon Sensor on unsupported OpenSUSE Tumbleweed by resolving the missing OpenSSL 1.1 dependency.
date: 2025-04-03T11:51:56+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

Crowdstrike's Falcon sensor is packaged for a number of distributions, including SUSE Enterprise Linux, but not for OpenSUSE Tumbleweed.  This is how I got it going. I suspect this would work on other unsupported distributions, possibly including Leap.

## Solution

Crowdstrike falcon current documentation for Linux can be found here: 
https://www.crowdstrike.com/tech-hub/endpoint-security/installing-falcon-sensor-for-linux/

For OpenSUSE Tumbleweed, install the latest release described as “SLES 15” but not as “IBM zLinux”. 

`zypper install` will fail because it depends on `libopenssl1_1`, which doesn't exist in Tumbleweed's repositories anymore.  I was able to locate packages here (just chose a source, Expert download, binary packages): https://software.opensuse.org/package/libopenssl1_1  . The file I downloaded was `libopenssl1_1-1.1.1w-13.1.7.6.x86_64.rpm` and this continues to work well with Falcon Sensor releases as of early 2026. 

Both the openssl and falcon packages are unsigned so install them both with `zypper install -i` or choose 'i' when prompted. 

Then attach it to your Crowdstrike account:

`sudo /opt/CrowdStrike/falconctl -s --cid=<cid>`

Where the CID is copied from the Sensor Downloads page in the Crowdstrike Dashboard.

Start the sensor:

`sudo systemctl start falcon-sensor`

Verify it's running:

`ps auxwww|grep falcon`

Finally, verify the host is online in Host Dashboard. 
