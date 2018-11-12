---
title: History of Nintendo Development
categories:
  - Software
tags:
  - Nintendo
---

Start no devkit everything basically bare metal
To having some libraries with common functions
To a sort of kernel being compiled in
To full blown OS
To evolution of the OS adapted for new hardware

Resources

* NES - 10/18/1985
    * Bare metal
    * https://wiki.nesdev.com/w/index.php/CPU_power_up_state
* Game Boy - 04/21/1989
    * http://gbdev.gg8.se/wiki/articles/Gameboy_Bootstrap_ROM
    * Bare metal
https://realboyemulator.wordpress.com/2013/01/03/a-look-at-the-game-boy-bootstrap-let-the-fun-begin/
http://www.its.caltech.edu/~costis/cgb_hack/gbc_bios.txt
* Super NES - 08/23/1991
    * https://archive.org/details/SNESDevManual
    * Bare metal
    * ROM

Bare metal
———————————————
Primitive static library kernels or just shared libraries for common things. Basic BIOS

* Nintendo 64 - 06/23/1996
    * https://n64squid.com/homebrew/n64-sdk/nusystem/
    * https://n64squid.com/homebrew/n64-sdk/development-hardware/
    * https://level42.ca/projects/ultra64/Documentation/man/pro-man/pro06/index.html
    * http://ultra64.ca/resouorces/documentation/
    * Small real time preemptive kernel. Supplied as a set of runtime libraries
    * PIF Rom - http://www.emulation64.com/ultra64/bootn64.html
    * http://www.jax184.com/projects/ultra64/Nintendo%20Ultra64%20Programming%20Manual+Addendums.pdf
    * http://ultra64.ca/resouorces/software/
    * http://ultra64.ca/files/software/other/sdks/n64sdk.7z
    * 
* Game Boy Advance - 03/21/2001
    * gbadev.org
    * http://problemkaputt.de/gbatek.htm#biosfunctions
    * devkitpro
    * BIOS functions but
https://mgba.io/2017/06/30/cracking-gba-bios/
* GameCube - 09/14/2001
    * First one that did something when turned on with no game
    * http://hitmen.c02.at/files/yagcd/yagcd/chap18.html
    * Dolphin OS
    * IPL contains OSIinit, alarms, memory, dvd
* Nintendo DS - 11/21/2004
    * Also gbadev.org
    * https://corgids.wordpress.com/2017/07/28/booting-the-nintendo-ds-a-technical-summary/
    * ARM7 and ARM9
    * https://media.ccc.de/v/23C3-1552-en-nintendo_ds
    * http://pineight.com/ds/pass/
    * https://gbatemp.net/threads/tutorial-how-to-check-your-firmware-version-on-ds-ds-lite.456211/
    * https://en.wikipedia.org/wiki/Nintendo_DS_homebrew (mentions different firmware versions)
    * ?????

Console that give exclusive access to the game
—————————————————————————————
Consoles that keep system services running after game launch

* Wii - 11/19/2006
    * wiibrew.org
    * https://hackmii.com/2009/06/ios-history-build-process/
    * Starts to have system services
    * Games ran with full hardware access
    * http://wiibrew.org/wiki/IOS
    * http://wiibrew.org/wiki/IOS/Syscalls
    * PPC with ARM9 http://wiibrew.org/wiki/Hardware/Starlet
    * https://media.ccc.de/v/25c3-2799-en-console_hacking_2008_wii_fail
    * 
* Nintendo DSi - 11/1/2008
    * http://dsibrew.org/wiki/Main_Page
    * https://gbatemp.net/threads/release-twltool-dsi-downgrading-save-injection-etc-multitool.393488/
    * http://problemkaputt.de/gbatek.htm
    * ARM9 and ARM7
    * ????
* Nintendo 3ds - 02/26/2011
    * 3dbrew.org
    * Lots of system services
    * Robust IPC to system services
    * https://www.3dbrew.org/wiki/Services_API
    * https://www.reddit.com/r/3dshacks/comments/6iclr8/a_technical_overview_of_the_3ds_operating_system/
    * ARM11 and ARM9
    * https://github.com/d0k3/Decrypt9WIP
    * https://github.com/AuroraWright/Luma3DS
    * https://media.ccc.de/v/32c3-7240-console_hacking
        * Derrick claims this is nintendo’s first kernel (doesn’t count wii? says there was something running on the coprocessor but not written by nintendo)
        * Keys shared between 3ds and WiiU
    * http://darthsternie.bplaced.com/3ds.html
    * https://github.com/d0k3/GodMode9
* Wii U - 11/18/2012
    * I assume this is basically like a Wii?
    * http://wiiubrew.org/wiki/Main_Page
    * http://wiiubrew.org/wiki/Cafe_OS
    * https://gbatemp.net/threads/ida-stuff.424786/
    * PPC with ARM9 on graphics chip running stuff in the background
* Nintendo Switch - 03/03/2017
    * switchbrew.org
    * reswitched.tech
    * Seems close to the 3ds model
    * http://switchbrew.org/index.php?title=TSEC
    * https://github.com/dekuNukem/Nintendo_Switch_Reverse_Engineering
    * https://media.ccc.de/v/34c3-8941-console_security_-_switch#t=716 (custom microkernel called Horizon)


https://en.wikipedia.org/wiki/Game_development_kit#Nintendo_Entertainment_System







