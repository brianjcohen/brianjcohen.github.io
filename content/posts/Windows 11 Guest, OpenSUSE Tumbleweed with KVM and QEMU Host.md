---
title: Windows 11 Guest, OpenSUSE Tumbleweed with KVM and QEMU Host
description: Running Windows 11 as a KVM/QEMU virtual machine on OpenSUSE Tumbleweed, including setup, networking fixes, and performance tips.
date: 2025-05-01T09:10:52+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

Why do I keep doing this to myself?  For as long as I can remember, for the 3 or 4 times I year I absolutely *must* do something in Windows, I say to myself: I'm going to get Windows running in a virtual machine on my Linux desktop. That way it's always there when I need it.  All I should have to do is install Virtualbox (or VMWare), install, and it should be good! Simple.  

It's never simple.

Both those solutions involve external kernel modules that have to be compiled every time you upgrade your kernel. Development of the drivers are not always up to date with the latest kernel that's being shipped with your distribution, which is especially a problem if you run a rolling distribution like Tumbleweed.  

Enter KVM and QEMU. Using kernel virtualization I should be able to get great performance with no external modules. No muss, no fuss. Simple.

It's never simple. 

Here's what I did. 

## Get Windows

I purchased a licensed copy of Windows 11 at https://www.microsoft.com/en-us/d/windows-11-home/dg7gmgf0krt0/0008, making sure to choose the "Download" option. After tax it came to $150.  Yes, I know you can probably get a copy of Windows for free and get it "activated" in some crooked way. No, I do not recommend you do this.   The file will download in ISO format  Make sure you make note of your Product Key on the confirmation page (although if you don't, it will also be in your confirmation email).


## Install All The Things

There are two ways to do this:

Go to yAST Control Center, Virtualization, Install Hypervisor and Tools, check the boxes next to KVM server and KVM tools and hit Accept.  yAST won't exist forever (slated to replaced with something called Cockpit?) and I'm more of a CLI guy anyway, so the alternative:

`sudo zypper install libvirt qemu virt-manager libvirt-daemon-driver-qemu qemu-kvm`

Now add yourself to the newly setup groups so that you can connect as a regular user.

`sudo usermod -aG libvirt,kvm your_username`  

Obviously replace `your_username` with your Linux login name.

Also, there's a magic variable that has to be in your environment:

`echo "export LIBVIRT_DEFAULT_URI='qemu:///system'" >> ~/.bashrc`

 Now, **reboot**.

## VM Setup

I can't believe I'm doing this, but honestly this post here is much better than anything I could write up. Just follow this:

https://sysguides.com/install-a-windows-11-virtual-machine-on-kvm

If you're having trouble finding the VirtIO drivers, they are here:

https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md


## If Networking is Broken

On OpenSUSE Tumbleweed, for me, there was no Internet access for the guest no matter what I tried, until I found a very helpful tidbit in the superuser.com link cited in References.  I had to set `firewall_backend = "iptables"` in `/etc/libvirt/network.conf` and then restart libvertd:

```
$ sudo systemctl restart libvirtd
```

## If you really want to bypass online activation

In theory that should make it so that your VM has Internet access, and Setup should be able to make it all the way through.  In practice I actually didn't figure that fix out until later in the game, and Windows Setup actually stopped me at some point because it couldn't proceed without network activation. To work around that, on the "Let's connect you to a network" screen, I pressed Shift + F10 to raise a Command Prompt.  Then I typed OOBE\BYPASSNRO and hit enter. This will reboot the VM.  When Setup makes it back to that same page, this time it will have a button that says "I don't have Internet".  Click that and it will let you proceed by making a local account instead of signing into a Microsoft account.  Once Windows is running, you can proceed with installing the virtio drivers.

## Setup a desktop launcher directly to your Windows guest

Create a file `~/.local/applications/windows-qemu.desktop`:

`[Desktop Entry]`  
`Name=Windows 11`  
`Comment=Launch Windows VM`  
`Exec=virt-manager --connect qemu:///system --show-domain-console windows`  
`Icon=virt-manager`  
`Terminal=false`  
`Type=Application`  
`Categories=System;Utility;Virtualization;`

## Ah Crap

A few weeks after getting this running, after many reboots and an upgrade of QEMU, I tried firing up my VM and I got this:

`Error starting domain: Requested operation is not valid: network 'default' is not active`

`Traceback (most recent call last):`
  `File "/usr/share/virt-manager/virtManager/asyncjob.py", line 88, in cb_wrapper`
    `callback(asyncjob, *args, **kwargs)`
  `File "/usr/share/virt-manager/virtManager/asyncjob.py", line 124, in tmpcb`
    `callback(*args, **kwargs)`
  `File "/usr/share/virt-manager/virtManager/libvirtobject.py", line 83, in newfn`
    `ret = fn(self, *args, **kwargs)`
  `File "/usr/share/virt-manager/virtManager/domain.py", line 1479, in startup`
    `self._backend.create()`
  `File "/usr/lib/python2.7/site-packages/libvirt.py", line 1062, in create`
    `if ret == -1: raise libvirtError ('virDomainCreate() failed', dom=self)`
`libvirtError: Requested operation is not valid: network 'default' is not active`

I resolved it this way:

`virsh net-autostart default`
`virsh net-start default`

(do not run those as root)

## Ah Crap, Part 2

If you're reading this, you are (like me) probably a desktop Linux user who occasionally needs Windows. That means you probably boot up the Windows guest somewhat, or very, infrequently.

Windows assumes it's your primary OS, installed on your hard disk, always running. As a result it performs a lot of background work in the, uh, background while you, uh, work. For speed, you know. 

That means if it's been off for weeks, when you boot it back up, it's going to be "time" for a lot of deferred background tasks to run. Like built-in malware detection and Windows Update. 

This will peg your guest, and host, CPU for a while. Your PC's fan will probably sound like an FA/18 Hornet.  In Windows Task Manager, you'll probably see "Background Task Host" eating 90% CPU. That will eventually give way to "Service Host: Windows Update".  Just give it time, it will finish and calm down. 

## Ah Crap, Part 3

Some extra stuff I've done:

* On the Windows 11 guest:
	* The sysguides reference has you turning off everything in "Performance Options" by choosing "Adjust by best performance".  I actually re-enabled "Smooth edges of screen fonts" because otherwise you lose anti-aliasing and everything looks bad.
	* I found that one or more rogue extensions were causing my browser (Brave) to consume high CPU on the guest. I disabled or removed all the extensions I didn't need, which resolved that problem.
	* I found the weather and news "widgets" were sluggish and pointless and using unnecessary memory and CPU so I disabled them by right-clicking the Taskbar > Taskbar Settings > Uncheck "Widgets". I also unchecked "Task View" and set "Search" to "Hide" while I was at it. As far as I'm concerned, Windows peaked in 2007 and I'm doing my best to return it to that. 
* On the host:
	* You can enable 3D graphics acceleration, however it will disable the ability to "suspend" the VM via Power Off > Save.  To set this up, while the guest is off, in the guest configuration, under "Display Spice" change Listen Type to "None" and check "OpenGL". Then under Video Virtio, enable "3D Acceleration". 

## References

https://superuser.com/questions/1671932/unable-to-connect-to-internet-in-windows-10-vm-using-kvm-qemu

https://cubiclenate.com/2019/06/11/virtual-machine-manager-with-qemu-kvm-on-opensuse-tumbleweed/ 

https://sysguides.com/install-kvm-on-linux

https://sysguides.com/install-a-windows-11-virtual-machine-on-kvm