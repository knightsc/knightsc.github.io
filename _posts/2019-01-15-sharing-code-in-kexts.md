---
title: Sharing code in KEXTs
categories:
  - Software
tags:
  - C
  - Kernel
  - macOS
---

Recently while reverse engineering three kernel extensions from a macOS security product, I noticed that there was a lot of duplicated code between all of the KEXTs. Common things like logging or initialization of data structures were clearly the same in each one. With an IOKit KEXT you can create parent classes in one KEXT and inherit from it in another one. In this case, these three extensions were not IOKit drivers. Although Apple doesn't document it very well you can share code across multiple KEXTs. This post covers some examples from Apple of shared KEXTs as well as how you would do it in your own KEXT.

# What Apple KEXTs have shared code?

There are over 400 different KEXTs installed in `/System/Library/Extensions` on a stock macOS machine. Each KEXT contains an `Info.plist` file that specifies information about the extension as well as it's depedencies. Every KEXT depends on the kernel and one or more of the various `kpi` pseudo KEXTs. Lots of them also depend on each other though. I wrote a small python script to generate a [GraphML document](https://gist.github.com/knightsc/5741e32ce5c59c1e0dbf30fd1ff8ac4b){: target="_blank"} of all these relationships and if you graph them you'll get something like this:

![KEXT Graph](/images/sharing-code-in-kexts-1.png){: .align-center}

Outside of the kernel, the biggest depedencies are all the different IOKit family extensions like: `IOPCIFamily`, `IOUSBHostFamily`, `IOHIDFamily`, `IONetworkingFamily`, `IOStorage` and others. But if we look closer at some of the extensions that aren't from IOKit we see that there are still some extensions that provide shared code to a lot of different KEXTs. 

For example the `corecrypto` extension has common hash and encryption functions exported from it and it gets used by all of the following KEXTs:

* `apfs`
* `AppleBCMWLANCore`
* `AppleCredentialManager`
* `AppleFDEKeyStore`
* `AppleKeyStore`
* `AppleMobileFileIntegrity`
* `AppleSEPManager`
* `CoreTrust`
* `Dont Steal Mac OS X`
* `IO80211Family`
* `IO80211FamilyV2`
* `IOImageLoader`
* `smbfs`

If we zoom in on the graph above and just highlight items related to `corecrypto` we can see this same list of extensions:

![Core Crypto Graph](/images/sharing-code-in-kexts-2.png){: .align-center}

# How do you share code between KEXTs?

So, if you want to write your own KEXT that has common code for other KEXTs to use, how do you go about it? Well it's actually easier than you think. By default any function you define is going to be visible in the global kernel name space. Apple actually recommends the following in the outdated [Kernel Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/style/style.html){: target="_blank"}

>Declare all functions and (global) variables static where possible to prevent them from being seen in the global namespace. If you need to share these across files within your KEXT, you can achieve a similar effect by declaring them \__private_extern__.

This is something that I think a lot of KEXT authors don't think to do. This is great for reverse engineering though because it means that most functions still have a readable name with them when looking at the disassembly.

Here's an example of a `Shared` KEXT with one function visible to other KEXTs and another function only available internally:

```c
#include <libkern/libkern.h>
#include <mach/mach_types.h>

int
Shared_Exported()
{
    return 5;
}

static int
Shared_NotExported()
{
    return 10;
}

kern_return_t
Shared_start(kmod_info_t *ki, void *d)
{
    int i = Shared_NotExported();
    printf("Shared_start: %d\n", i);
    return KERN_SUCCESS;
}

kern_return_t
Shared_stop(kmod_info_t *ki, void *d)
{
    return KERN_SUCCESS;
}
```

The `Shared_Exported()` function will now be visible to other KEXTs. The only other thing you need to do is add an `OSBundleCompatibleVersion` key to the KEXTs `Info.plist` like this:

```xml
<key>OSBundleCompatibleVersion</key>
<string>1.0.0</string>
<key>OSBundleLibraries</key>
<dict>
    <key>com.apple.kpi.libkern</key>
    <string>8.0.0</string>
</dict>
```

This allows you to declare the earliest version number of your API other KEXTs can depend on. Essentially this allows you to indicate how backwards compatible your shared code is. Without specifying this the KEXT loading mechanism won't be able to properly load your KEXT dependency.

Here's an example of a different KEXT that calls the `Shared_Exported()` function:

```c
#include <libkern/libkern.h>
#include <mach/mach_types.h>

extern int Shared_Exported(void);

kern_return_t
Driver_start(kmod_info_t *ki, void *d)
{
    int i = Shared_Exported();
    printf("Driver_start: %d\n", i);
    return KERN_SUCCESS;
}

kern_return_t
Driver_stop(kmod_info_t *ki, void *d)
{
    return KERN_SUCCESS;
}
```

This time in the `Info.plist` you just specify your own custom depedency and what version of the KEXT API you need:

```xml
<key>OSBundleLibraries</key>
<dict>
    <key>sc.knight.kext.Shared</key>
    <string>1.0.0</string>
    <key>com.apple.kpi.libkern</key>
    <string>8.0.0</string>
</dict>
```

You can download a full Xcode project with these two KEXTs from here:

[https://github.com/knightsc/osx_and_ios_kernel_programming/tree/master/Examples/SharedCodeExample](https://github.com/knightsc/osx_and_ios_kernel_programming/tree/master/Examples/SharedCodeExample){: target="_blank"}

After compiling both KEXTs you will need to do the following:

* `sudo chown -R root:wheel Shared.kext`
* `sudo chown -R root:wheel Driver.kext`
* `sudo cp Shared.kext /Library/Extensions/`
* `sudo kextload Driver.kext`

If you don't see any errors then you did everything correctly. You can check on the example driver with the following two commands:

```bash
$ sudo kextstat | grep -v apple
Index Refs Address            Size       Wired      Name (Version) UUID <Linked Against>  
  155    1 0xffffff7f82997000 0x3000     0x3000     sc.knight.kext.Shared (1) 1AB1D921-83D3-3226-936B-EFE323F731C4 <4>
  156    0 0xffffff7f8299a000 0x2000     0x2000     sc.knight.driver.Driver (1) 9675791F-B76B-3EA9-8F05-FED42F38EFC4 <155 4>

$ sudo dmesg
...
Shared_start: 10
Driver_start: 5
...
```

# When should you share code between KEXTs?

So now the real question, when should you create multiple interdependant KEXTs? I think standard good coding practices apply here. So if you only have a single KEXT and you think you might create more it's probably not time to create a KEXT of shared code yet (Although I have seen some commercial products that do this). More KEXTs mean more hassels for your users. As of macOS 10.13, third party KEXTs need to be approved by the user before they can run or alternatively whitelisted by your MDM server. If you have two or three or more KEXTs and they all have duplicated code then it might be time to think about refactoring it out into a shared KEXT. Even in this case, you should still be asking yourself if it's really worth it. You could always take the approach of simply having an Xcode project with a single source file shared in all your different KEXTs. This would still reduce the code duplication but not require the installation of an additional KEXT.
