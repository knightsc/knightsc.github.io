---
title: D-Link DPH-128MS (Part 1)
categories:
  - Reverse Engineering
tags:
  - Linux
  - MIPS
---

Alright, so at work a couple years back we got these fancy new VOIP phones. I immediately set into trying to figure things out to see what else we might do with them. I unfortunately set that project aside but just recently picked it back up. I figured I would post the info I found out in case anyone else was interested. I decided to split this into multiple posts in an attempt to keep the posts a little shorter. Anyways this first post is about the firmware.

<img class="alignnone" title="DPH-128MS" src="http://farm9.staticflickr.com/8297/7988303892_a79a527ef7_n.jpg" alt="" width="320" height="271" />

So D-Link at one point made DPH-125MS and DPH-128MS phones that used Microsoft Response Point technology. As of right now, their website lists them as "phased out". Since my phone is a DPH-128MS these posts talk specifically about that, but the firmware is similar and I'm sure the phones are similar hardware. The latest firmware can be downloaded from here:

<a href="ftp://ftp.dlink.com/VoIP/dph128MS/Firmware/dph128MS_firmware_011510.zip">DPH-128MS Firmware v01.15.10</a>

If you unzip this, inside the folder you'll find an interesting file called Firmware.xml and in there it says that it upgrades via tftp. I confirmed this by watching network traffic between my computer and the phone while upgrading. The xml file also says that full.img is the actual firmware file. So the first thing I did was to run strings on full.img. One of the first strings that I actually came across was this:

```
/home/mikko/release_p125/kernel/linux-2.4.17_mvl21/include/linux/dcache.h
```

Alright great, I know linux. Not only that, but it tells us what kernel it's running and based on the mvl21 part we know it's MontaVista Linux. Anyway the next thing I did was fire up a hex editor looking for magic numbers. That's the next easiest thing to do. The very first number I found was `0x27051956`. A quick google search says this is the magic number for u-boot. Quickly skimming over the u-boot format I can at least see that there's two different things in this file. Something called `ACT_Image_SD_P125_MSFT` and something else called `P125_MSFT_Kernel_Image`.

Anyway rather than looking more into u-boot i just kept searching for magic numbers. Just sorta glancing through the file for spots where it looked like some data ended and other stuff started. I eventually also found the magic number for gzip and the magic number for jffs2.

Now here's where everything gets a little fuzzy since I did this part when we originally got the phones. Basically though I just used the hex editor and manually extracted out the gzip part and the jffs2 part. The gzip part was clearly a ramdisk and most likely was a ramdisk embedded in the kernel image. I quickly checked out the ramdisk

```shell
gunzip ramdisk.gz
mkdir ramdisk_dir
mount -o loop ramdisk ramdisk_dir
```

Nothing terribly interesting, a bare bones linux system. Busybox, the usual stuff for small embedded systems. Next I set about checking out the jffs2 filesystem.

```shell
mkdir m
modprobe mtdram total_size=24576 erase_size=128
cat /proc/mtd
modprobe mtdblock
dd if=jffs2.img of=/dev/mtdblock0
mount -t jffs2 /dev/mtdblock0 m
```

Now this is the image that was interesting. There was only a handful of files but the two that caught my eye were `act_sip` and `tftpsrv`. The  `act_sip` seemed to most likely be the main voip app on the phone. `tftpsrv` was the app that I decided to focus my attention on since I was interested in possibly uploading new firmware.

Aaaand this is where I'll stop part one. Now mind you, if I had of realized this was a u-boot image in the first place I probably could have just unpacked things and created a new ramdisk with some extra stuff and re-flashed the phone and as you'll soon find out I might still have to do this. But I didn't, I set off checking out `tftpsrv` and that's what I'll talk about in the next post.
