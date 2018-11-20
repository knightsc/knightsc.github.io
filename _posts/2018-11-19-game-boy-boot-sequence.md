---
title: Game Boy boot sequence
categories:
  - Reverse Engineering
tags:
  - Assembly
  - Nintendo
  - Z80
---

The Nintendo Game Boy was first released in North America on April 21, 1989. It wasn't the most powerful handheld of the time but certainly was the most popular. Over the years I've done some reverse engineering on Game Boy Advance and Nintendo DS handhelds but have never looked at the Game Boy. I thought it might be interesting to take a closer look into the boot ROM of the Game Boy and what it does at start up.

# Overview

When the Game Boy starts up the boot ROM is mapped over cartridge memory from `$0000` - `$00ff`. Execution starts directly from `$0000`. What follows is a high level overview of what the boot ROM does:

1. Clear VRAM from `$8000` - `$9fff`, setting it to zero
2. Enable sound and configure sound channels
3. Set up the background palette data (Color 0 is white everything else is black)
4. Load the cartridge header logo tiles into VRAM starting at `$8010`
5. Set up the background tile maps
6. Scroll the cartridge header logo down the screen
7. Play the startup sound
8. Compare the cartridge logo to the boot ROM logo
9. Loop forever if the logos don't match
10. Check the cartridge header checksum
11. Loop forever if the header checksum is invalid
12. Disable the boot ROM
13. Continue execution from the cartridge ROM

I think that these last two steps are important and some times misunderstood. I've seen some write ups that describe the Game Boy boot ROM as jumping to the cartridge entry point which is incorrect. When the system starts the boot ROM is actually covering up the first `$ff` bytes of the cartridge ROM. Once the boot ROM is disabled the full cartridge ROM address space is visible and execution simply continues running from `$0100` which is the next instruction after the boot ROM disable at `$00fe`.

As you can see, the boot process for the original Game Boy is fairly straight forward. The whole boot ROM is only 256 bytes long! It's whole purpose is to show the logo from the cartridge and then validate the cartridge logo and header checksum. The idea was that if you were an unlicensed Nintendo developer and you produced an unlicensed game you would have to reproduce Nintendos logo which is a registered trademark. This would in turn allow Nintendo to manually enforce anti-piracy measures through litigation.

# Tools and Documentation

Game Boy hardware reference  
[http://bgb.bircd.org/pandocs.htm](http://bgb.bircd.org/pandocs.htm){: target="_blank"}

Game Boy disassembler  
[https://github.com/knightsc/hopper/tree/master/cpus/GBCPU](https://github.com/knightsc/hopper/tree/master/cpus/GBCPU){: target="_blank"}

Commented disassembly  
[dmg_rom.asm](https://gist.github.com/knightsc/ab5ebda52045b87fa1772f5824717ddf){: target="_blank"}

Game Boy assembler  
[https://github.com/rednex/rgbds](https://github.com/rednex/rgbds){: target="_blank"}
