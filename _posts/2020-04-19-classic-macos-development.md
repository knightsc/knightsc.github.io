---
title: Classic Mac OS development
categories:
  - Software
tags:
  - 68k
  - Assembly
  - macOS
  - Pascal
---

Before macOS, and before OS X, there was just Mac OS. This is often referred to as "Classic" Mac OS. It includes System 1 all the way up to Mac OS 9.x. I started using a Mac with System 6 on a Macintosh Classic. Then I moved up to a Macintosh IIsi running System 7. Finally, after the PowerPC transition, I used a Power Macintosh 8500 which ran all of the later versions of "Classic" Mac OS. I was recently having a conversation with another developer who grew up using Macintosh computers and we were both reminiscing about some of our early development experiences on Mac. While System 6 was the first Mac OS version I used, I didn't start really writing Mac apps until the Mac OS 8 era. This got me thinking that it might be interesting to spend some time re-learning "Classic" Mac OS app development.

As I mentioned previously I didn't really start programming until Mac OS 8 and by then CodeWarrior had solidly cemented itself as the IDE of choice for Mac developers. I decided for this exploration that I wanted to stick to early Mac software as much as possible. I chose to only look for tools that were available for Mac prior to the 1990s.

# Emulation

Since I no longer have any physical "Classic" Mac hardware I decided to turn to emulation. I'll go over some of the more populator emulators and why I chose the one I did.

## SheepShaver

[https://sheepshaver.cebix.net/](https://sheepshaver.cebix.net/){: target="_blank"}

SheepShaver emulates a Power PC Macintosh. It was originally created for BeOS back in 1998. Since then, it has become an open source project. It's capable of running Mac OS 7.5.2 through 9.0.4. If you're interested in running the more recent versions of "Classic" Mac OS this is probably the emulator you should choose. Mac OS 7.5.2 was released in 1995 and in turn SheepShaver doesn't fit my criteria of sticking to software and tools available prior to the 1990s.

## Basilisk II

[https://basilisk.cebix.net/](https://basilisk.cebix.net/){: target="_blank"}

Basilisk II emulates a 68k Macintosh. Originally released in 1997 by the same developer as SheepShaver. It's capable of running up to Mac OS 8.1. This is another very popular emulator and a lot of people looking to emulate 68k Macintoshes choose this one. It is also open source, however it is no longer being maintained.

## Mini vMac

[https://www.gryphel.com/c/minivmac/](https://www.gryphel.com/c/minivmac/){: target="_blank"}

Mini vMac is a spinoff of the [vMac](http://www.vmac.org/){: target="_blank"} project. It also emulates a 68k Macintosh. It has a focus on the early Macs with the default build emulating a Macintosh Plus. Mini vMac is capable of emulating up to Mac OS 7.5.5. It's also open source and unlike Basilisk II is still being maintained.

So what's the difference between Mini vMac and Basilisk II? The FAQ page for Mini vMac has a great explanation.

>The biggest current difference is that Mini vMac emulates the earliest Macs, while Basilisk II emulates later 680x0 Macs. The fundamental technical difference is that Basilisk II doesn’t emulate hardware, but patches the drivers in ROM, while Mini vMac emulates the hardware (with the exception of the floppy drive).
>
>The consequences are that some of the earliest Mac software will run in Mini vMac and not Basilisk II, while much of the later software will run in Basilisk II and not Mini vMac. For software that will run in either, the emulation in Mini vMac can be more accurate, while Basilisk II offers many more features (including color, larger screen, more memory, network access, and more host integration).
>
>Mini vMac aims to stay simple and maintainable. So Mini vMac only has compile time preferences, where as Basilisk II has many run time preferences. And Mini vMac uses a rather simple emulation of the processor, compared to Basilisk II, which could make Mini vMac slower. 

The fact that Mini vMac focuses on early Macs and ealy Mac software it fit my criteria well. It has a good [Getting Started](https://www.gryphel.com/c/minivmac/start.html){: target="_blank"} page as well as a collection of other [Tutorials](https://www.gryphel.com/c/minivmac/recipes/index.html){: target="_blank"} to help you get system software and get up and running. I went through all of the tutorials and now have a working emulated Mac Plus running System 6.0.8.

![Mini vMac Screenshot](/images/classic-macos-development-1.png){: .align-center}

# Software

With an emulator up and running I next needed to find software. Luckily, there are a few sites that host repositories of software for old Mac OS versions. The following sites have been some of the most helpful in terms of finding old software:

* [https://www.gryphel.com/c/sw/index.html](https://www.gryphel.com/c/sw/index.html){: target="_blank"}
* [https://macintoshgarden.org/](https://macintoshgarden.org/){: target="_blank"}
* [https://www.macintoshrepository.org/](https://www.macintoshrepository.org/){: target="_blank"}

# IDEs

I mentioned earlier that CodeWarrior was the IDE of choice when I started Mac development but since it came out in the 90s it didn't fit my criteria for early Mac development. Additionally while C/C++ had become the language of choice for the Mac in the 90s, back in the 80s Pascal was by far more common. I also needed an IDE that supported System 6.

While looking for Pascal compilers I came across two main contenders: Borland Turbo Pascal and THINK Pascal. Both seemed like good potential candidates. They had versions that came out in the late 80s and supported System 6. THINK Pascal seemed to be fairly popular during the era.

An alternative, that I had used a handful of times before CodeWarrior, was the Macintosh Programmer's Workshop (MPW). MPW was the development environment provided by Apple. In the 80s it was quite expensive. It had a 68k assembler, a pascal compiler, and (new for MPW 2.0) a C compiler as well. This seemed like a fun choice because of the range of languages supported but also because it was the official offerring provided by Apple. After downloading MPW 2.0 from the software links above I had a working development environment.

![Mini vMac Screenshot](/images/classic-macos-development-2.png){: .align-center}

# Books

The last thing I needed were some good programming books from the time period. I found a wonderful resource in the Vintage Apple website.

[https://vintageapple.org/macprogramming/](https://vintageapple.org/macprogramming/){: target="_blank"}

Here's a list of the books I've found most useful so far:

* [Inside Macintosh Volume I](https://vintageapple.org/inside_o/pdf/Inside_Macintosh_Volume_I_1985.pdf){: target="_blank"}
* [Inside Macintosh Volume II](https://vintageapple.org/inside_o/pdf/Inside_Macintosh_Volume_II_1985.pdf){: target="_blank"}
* [Inside Macintosh Volume III](https://vintageapple.org/inside_o/pdf/Inside_Macintosh_Volume_III_1985.pdf){: target="_blank"}
* [Inside Macintosh Volume IV](https://vintageapple.org/inside_o/pdf/Inside_Macintosh_Volume_IV_1986.pdf){: target="_blank"}
* [MPW and Assembly Language Programming for the Macintosh](https://vintageapple.org/macprogramming/pdf/MPW_and_Assembly_Language_Programming_for_the_Macintosh_1987.pdf){: target="_blank"}
* [Programming with Macintosh Programmers Workshop](https://vintageapple.org/macprogramming/pdf/Programming_With_Macintosh_Programmers_Workshop_1987.pdf){: target="_blank"}

Inside Macintosh Volumes I - III cover everything you would ever want to know about the early Mac and how it worked. It also covers all of the OS managers and their API's as well. Inside Macintosh Volume IV covers changes for the Macintosh Plus, which is helpful since Mini vMac emulates a Macintosh Plus. The other two books have some good information about MPW itself and how it works as well as some okay intro to Mac programming.

# What's Next?

With an emulated Mac configured and an IDE chosen I've started to write some little test programs in Pascal. While I've never written a Mac program in Pascal, I have written many Delphi applications on Windows. I've also started to search out some old Mac viruses from the 80s to take a look at how they worked. Overall, I find it a nice change of pace to be able to boot into System 6, do some coding, play some old games and remember a time when computers were a lot less complicated to use.
