---
title: macOS kernel coding style
categories:
  - Software
tags:
  - C
  - C++
  - Kernel
  - macOS
---

I've recently been doing some work reverse engineering IOKit drivers. When I approach reverse engineering a binary for a system or language that I'm not familiar with I like to start by finding some similar example code. This gives me an opportunity to sit down and read through the high level source code and wrap my head around how an engineer would approach writing the code in the first place. Then I can actually compile the example code and compare the disassembly with the high level source to really solidify what different code patterns and data structures look like in the disassembly. Something I noticed while reading through a lot of IOKit example code was that the coding style was wildly inconsistent. I thought it might be interesting to attempt to document a set of kernel coding styles for macOS based on what I've seen used in example code as well as in the XNU kernel source code.

Before deciding to create a style guide myself I first tried to see if anything already existed. After all, there's a great [Linux kernel coding style](https://www.kernel.org/doc/html/latest/process/coding-style.html){: target="_blank"} document, why shouldn't there be one for macOS as well? Unfortunately while Apple does have a lot of kernel code available as open source the development is not done in the open. This means that there's not really contributors outside of Apple and in turn not much reason for Apple to have a standard coding style defined for the community. I did manage to find one archived document from Apple that touches on kernel coding style called [Kernel Programming Style](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/style/style.html){: target="_blank"}. I also found `clang-format` rules in the [XNU source code](https://github.com/apple/darwin-xnu/blob/master/.clang-format){: target="_blank"}, which is helpful. Unfortunately it doesn't seem like these rules are always followed in the code bases out there.

So why define a macOS kernel coding style at all? Well for a couple reasons. First, I believe a consistent coding style makes code easier to read and understand. Additionally, standard coding style conventions can help avoid introducing some types of coding errors. Most importantly, I just personally would like to see more up to date and relevant kernel related information out there for macOS. So, what follows is my definition of a macOS kernel coding style guide. It doesn't strictly follow every convention used in official Apple source code but it is close. Feel free to reach out with constructive feedback, good or bad.

# Kernel Coding Style

## Indentation

This one is a no brainer to me. Out of the box Xcode has the following settings

Prefer indent using: Spaces
Tab width: 4 spaces
Indent width: 4 spaces
Tab key: Indents in leading whitespace

Some of the XNU source has identation as tabs and some has it as spaecs. The spaces win.

## Line length

The limit on the length of lines is 132 columns.

I think the most important thing when it comes to line length is to be consistent. Everyone has their own preference when it comes to line length. 132 columns is chosen simply because that's what the [`.clang-format`](https://github.com/apple/darwin-xnu/blob/master/.clang-format#L78){: target="_blank"} config files specify.

## Headers

You should have header guards. They should be named the name of the file with periods replaced as underscores.

HEADERNAME_H

File extensions should be .h
Yes I know Xcode will create C++ header files as .hpp but nobody follows that convention.

## Spaces

int *thing;

if (this)

function(int one, int two)
{

}

## Braces

if () {

}

Here's an area where I agree with linux. K&R know best. Braces should be at the end of the line with the one exception being functions or methods

function
{


}

## IOKit specific conventions

common conventions

IOKit

super

class definition

Yes your class needs to include names space com_company_

## Bundle IDs for KEXTs

com.<company>.driver.<Name> regular kernel module
com.<company>.iokit.<Name> iokit drivers
com.<company>.kext.<Name>  shared code

# Conclusion

I have a handful of example projects using this style guide up on github:

[https://github.com/knightsc/osx_and_ios_kernel_programming](https://github.com/knightsc/osx_and_ios_kernel_programming){: target="_blank"}
