---
title: AppImage made Easy
description: 
date: 2025-05-28T10:20:21+00:00
draft: false
tags:
  - linux
showTags: true
---
## Background

AppImage always struck me as a great idea in theory but a failure in practice.
v
The promise:  "Linux apps that run anywhere".  "Download an application, make it executable, and run! No need to install. No system libraries or system preferences are altered."  Both of these quotes are from appimage.org.

Reality of installing an AppImage

1. Download the .AppImage file. It ends up somewhere, maybe ~/Downloads
2. Launch a terminal.  
3. `cd Downloads; chmod u+x ./whatever-x.x.xx.AppImage`

Reality of running an AppImage

1. Launch a terminal.
2. `cd Downloads; ./whatever-x.x.xx.AppImage`

Do this every time you want to launch it. Keep the terminal window open because it's now your app's controlling terminal.  

All of this is annoying. If you're coming from Windows or Mac, the face-palming is almost automatic.

But wait, you say. That's ridiculous. Just setup a `.desktop` file so you can launch it from your GNOME, KDE Plasma, or XFCE launcher.   So basically:

1. Create ~/.local/share/applications/whatever.desktop
2. Write in something like this:

`[Desktop Entry]`  
`Name=Whatever App`  
`Comment=Launch Whatever App`
`Exec=/home/me/Downloads/whatever-x.x.xx.AppImage`
`Icon=`
`Terminal=false`  
`Type=Application`  
`Categories=System;Utility;`

Nevermind  that you don't have an icon (those are buried inside the appimage and you'd need to extract it with something called `appimagetool`) and you don't know what Categories to use.   If you were facepalming before, now you're screaming at the clouds in the sky.

But wait, it gets better. Want to upgrade your application because the vendor shipped a new version? Just manually download the latest version from their site and manually update the `.desktop` file.

It's worth noting that Windows 3.11 in 1993 was literally a better experience than this. 


## Enter appimaged

There are actually a few projects out there that purport to do this, but appimaged is the only one I've found that actually works. It helps that it was written by the inventor of the AppImage format itself.

https://github.com/probonopd/go-appimage

Oddly enough, there's really no mention of this daemon on appimage.org; all the Quick Start and related documentation there is telling you to go through the "set executable" and "run from the terminal" contortions described above. 

If you're running OpenSUSE Tumbleweed like I am, you're in luck. appimaged is in your repositories:

`sudo zypper install appimaged`

You will need to enable the systemd user service:

`systemctl enable --user appimaged.service`

And start it

`systemctl start --user appimaged.service`

I believe the installer will create a folder called `~/Applications` but if not, you should create it.

Now, just download AppImage files to that folder. That's it. appimaged will:

* Watch that folder
* Detect the new appimage
* Setup a `.desktop` file so it will appear in your launcher

It pretty magically handles upgrades too, it will update your launcher automatically if a new version of an AppImage is downloaded. Many apps have built-in self-update features (such as Cursor) which just download the new AppImage to the same directory, which in turn will trigger this functionality upon restart of the app.

If you delete an AppImage from your `~/Applications` folder, it will "unregister" that app and remove the `.desktop` file. 

And one last cool thing -- appimaged is, itself, actually distributed as an AppImage and you'll find it in `~/Applications` alongside everything else.  


