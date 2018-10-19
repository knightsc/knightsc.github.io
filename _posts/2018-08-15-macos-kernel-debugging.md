---
title: macOS Kernel Debugging
categories:
  - Debugging
tags:
  - Kernel
  - LLDB
  - macOS
  - Python
  - VMware
---

Getting started debugging the kernel can be very intimidating the first time you do it. There are various in depth guides that have been written over the years but they frequently get out of date. This post provides a short, concise, and up to date guide on getting started as easily as possible.

You can't debug the kernel of your own machine so you will either need a second physical machine or to use a virtual machine. The instructions below should work regardless of what type of target machine you're using. I personally use VMware Fusion.

# Setup

## Target
1. Run `sw_vers | grep BuildVersion` to determine your build version.
2. Download the appropriate Kernel Debug Kit from [Apple](https://developer.apple.com/download/more){: target="_blank"}.
3. Install the KDK package.
4. Reboot to the Recovery System by restarting your target machine and hold down the `Command` and `R` keys at startup.
5. From the Utilities menu, select `Terminal`.
6. Type, `csrutil disable` to disable System Integrity Protection.
7. `sudo reboot`
8. `sudo cp /Library/Developer/KDKs/<KDK Version>/System/Library/Kernels/kernel.development /System/Library/Kernels/`
9. `sudo kextcache -invalidate /Volumes/<Target Volume>`
10. `sudo nvram boot-args="debug=0x44 kcsuffix=development -v"`
11. `sudo reboot`

## Host
1. Install the KDK package.
2. `echo "settings set target.load-script-from-symbol-file true" >> ~/.lldbinit`

# Debugging

1. On the target machine hold down `Command`, `Option`, `Control`, `Shift` and `ESC` to trigger a NMI and break into the debugger.
2. Type `lldb -o "kdp-remote <target ip address>"` on the host machine to start lldb and connect.
3. `(lldb) b 0xffffff8000e838f8` set a breakpoint on an address.
4. `(lldb) b hfs_vnop_setxattr` set a breakpoint on a symbol.
5. `(lldb) continue`

# Uninstalling

To return to the Release kernel, simply perform the following steps on your target machine.

1. `sudo rm /System/Library/Kernels/kernel.development`
2. `sudo nvram -d boot-args`
3. `sudo rm /System/Library/Caches/com.apple.kext.caches/Startup/kernelcache.de*`
4. `sudo rm /System/Library/PrelinkedKernels/prelinkedkernel.de*`
5. `sudo kextcache -invalidate /`
6. `sudo reboot`

# Final Notes

If you're using a VM,   It can difficult to generate the NMI using the key combination mentioned above. An easy alternative to using the key combination is to use dtrace.
```
sudo dtrace -w -n "BEGIN { breakpoint(); }"
```
There are various kernel modules that provide alternate ways to drop into the debugger as well.

[https://github.com/knightsc/EnterDebugger](https://github.com/knightsc/EnterDebugger){: target="_blank"}  
[https://github.com/shiro-t/PseudoNMI](https://github.com/shiro-t/PseudoNMI){: target="_blank"}  

There are additional debug flags that can be set in the `boot-args` nvram variable. This is the current list of supported flags as of macOS 10.13.

<script src="https://gist.github.com/knightsc/619abdf9ca62602351b3aa2cce1b0704.js"></script>

# References

[Kernel Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming){: target="_blank"}  
[IOKit Device Driver Design Guidelines](https://developer.apple.com/library/archive/documentation/DeviceDrivers/Conceptual/WritingDeviceDriver){: target="_blank"}  
[https://reverse.put.as/2009/03/05/mac-os-x-kernel-debugging-with-vmware](https://reverse.put.as/2009/03/05/mac-os-x-kernel-debugging-with-vmware){: target="_blank"}  
[http://ddeville.me/2015/08/kernel-debugging-with-lldb-and-vmware-fusion](http://ddeville.me/2015/08/kernel-debugging-with-lldb-and-vmware-fusion){: target="_blank"}  
[https://lightbulbone.com/posts/2016/10/intro-to-macos-kernel-debugging](https://lightbulbone.com/posts/2016/10/intro-to-macos-kernel-debugging){: target="_blank"}  
[https://klue.github.io/blog/2017/04/macos_kernel_debugging_vbox](https://klue.github.io/blog/2017/04/macos_kernel_debugging_vbox){: target="_blank"}  
[https://www.mdsec.co.uk/2018/08/endpoint-security-self-protection-on-macos](https://www.mdsec.co.uk/2018/08/endpoint-security-self-protection-on-macos){: target="_blank"}  
