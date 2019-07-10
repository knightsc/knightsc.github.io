---
title: OSX/CrescentCore VM evasion
categories:
  - Malware
tags:
  - Assembly
  - LLDB
  - macOS
  - Swift
  - x86
---

[Intego](https://www.intego.com){: target="_blank"} recently shared information

[https://www.intego.com/mac-security-blog/osx-crescentcore-mac-malware-designed-to-evade-antivirus/](){: target="_blank"}

# OSX/CrescentCore

https://www.virustotal.com/gui/file/638004ee6a45903dcbf03d03e31d2e83c6270377973a64188f0b89d4062f321e/detection

# General approaches

## IOKit

This is the approach used `OSX/CrescentCore`. 

`ioreg -l | grep -e Manufacturer`

`OSX/MacKeeper`

`system_profiler SPHardwareDataType SPUSBDataType`


## cpuid

https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/osfmk/i386/cpuid.c#L1172

## Kernel Extensions

## SIP is disabled

## Look for weird internal things

Are syscalls hooked?

Does task_for_pid work when it shouldn't