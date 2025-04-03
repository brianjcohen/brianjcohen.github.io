---
title: "Running the Crowdstrike Falcon Sensor on OpenSUSE Tumbleweed"
description: 
date: "2025-04-03T11:51:56+00:00"
draft: false
---
Crowdstrike's Falcon sensor is packaged for a number of distributions, including SUSE Enterprise Linux, but not for OpenSUSE Tumbleweed.  This is now I got it going. I suspect this would work on other unsupported distributions, possibly including Leap.

Crowdstrike falcon current documentation for Linux can be found here: 
https://www.crowdstrike.com/tech-hub/endpoint-security/installing-falcon-sensor-for-linux/

For OpenSUSE Tumbleweed, install the latest release described as “SLES” but not as “IBM zLinux”

`zypper install` will fail because it depends on libopenssl1_1, which nothing provides for Tumbleweed.  I was able to locate packages here (just chose a source, Expert download, binary packages): https://software.opensuse.org/package/libopenssl1_1 

Both the openssl and falcon packages are unsigned so install them both with `zypper install -i` or choose 'i' when prompted. 

Then attach it to your Crowdstrike account:

`/opt/CrowdStrike/falconctl -s --cid=<cid>`

Where the CID is copied from the Sensor Downloads page in the Crowdstrike Dashboard.

Finally, verify the host is online in Host Dashboard. 
