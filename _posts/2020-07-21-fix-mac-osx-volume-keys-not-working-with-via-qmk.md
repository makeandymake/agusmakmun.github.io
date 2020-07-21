---
layout: post
title:  "FIX: Mac OSX Volume keys not working with VIA/QMK"
date:   2020-07-21 11:21:00 +0100
---

I recently finished building my own mechanical keyboard (Discipline 65) and was excited to get some media keys setup using VIA (the default firmware on this board). My plan was to configure the right most column for controlling the system volume plus brightness control. However, despite trying a number of different options including `KC__VOLUP` and `KC__VOLDOWN` which claim to be Mac specific, I just could not get the system volume controls working.

After a couple of hours of frustration I jumped on the official QMK Discord and asked there, getting a very quick answer that solved all my problems. It seems that for some reason, VIA doesn't support those Mac specific keycodes so just ignores them when you put them in. Here's how you fix it.

<center><img src="/static/img/via_screenshot.png" width="500" align="center"></center>

Instead of using the volume command, you use the "any" key which can be found in the "special" section of the keymapping area, then for Volume up, you enter the code `0x80` and for volume down you add `0x81` (incidently, if you want mute you need to use `0x7f`). 

Big thanks to the guys in the QMK/VIA discord, you were super helpful!
