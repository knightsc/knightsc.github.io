---
title: System Extension internals
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

One of the most exciting things announced at this years WWDC was [System Extensions](https://developer.apple.com/system-extensions/){: target="_blank"}. From a security perspective I think this is a really important advancedment for macOS. It means less third party code running in kernel space which should mean more security and stability. From a programmers perspective I think this is even more important. It means that the code developers previously had to write in C++ can now be written in a more modern language like Swift. Apple has been attempting to wrangle in kexts for a while now and this seems to be the final nail in the coffin. They have said macOS 10.15 will be the last release to fully support kexts without compromises and that in future releases of macOS Kernel Extensions with System Extension equivalents will not load at all. I thought it might be interesting to look at the internals of how System Extensions work.

First a disclaimer: everything that follows is based on the Catalina betas so these details might change by the time of the final release. Additionally my goal with this article is to provide an extremely high level overview. There are a lot of moving parts to System Extensions and it wouldn't be possible to cover all the internals in one post. With that out of the way let's take a look at how all the different subsystems communicate with each other.

![System Extension Internals](/images/system-extension-internals-1.png){: .align-center}

Conceptually there's really two big chunks to this. A user program calling into system services to activate or deactivate an extension and then the system itself running the extensions. The extensions then are what communicate with the kernel. Let's take a look at some of the different subsystems.

# SystemExtensions.framework
---

This is one of the new frameworks in Catalina. It's written in Objective-C and is pretty small. You can read through the official documentation on it [here](https://developer.apple.com/documentation/systemextensions){: target="_blank"}. Its sole purpose is to provide end user applications a way to install and uninstall system extensions. Installation is one area that I hope continues to evolve because for critical security extensions it doesn't make sense that a user application needs to run once before you're protected. Regardless there are a couple internal classes that are worth calling out.

## OSSystemExtensionInfo

This class represents the full information pertaining to a system extension. This object conforms to `NSSecureCoding` and is used when making requests to `OSSystemExtensionPointListener` instances.

## OSSystemExtensionPointListener

The main System Extension daemon has friend daemons that it uses to handle the different types of system extension categories. Both `endpointsecurityd`and `nesessionmanager` use this class at start up to create a XPC service that listens for system extension start and stop events.

## OSSystemExtensionClient

This class is the main way for the different applications to talk to `sysextd`. You can see a list of apps using this class by searching for it with ripgrep

```shell
$ rg -l -a -M 80 OSSystemExtensionClient 2> /dev/null
/sbin/kextload
/usr/bin/kextutil
/usr/bin/systemextensionsctl
/usr/libexec/endpointsecurityd
/usr/libexec/kextd
/usr/libexec/nesessionmanager
/usr/sbin/kextcache
/System/Library/Frameworks/SystemExtensions.framework/Versions/A/Helpers/sysextd
/System/Library/Frameworks/SystemExtensions.framework/Versions/A/SystemExtensions
/System/Library/PreferencePanes/Security.prefPane/Contents/MacOS/Security
/System/Library/PrivateFrameworks/DesktopServicesPriv.framework/Versions/A/DesktopServicesPriv
/System/Library/PrivateFrameworks/DesktopServicesPriv.framework/Versions/A/Resources/DesktopServicesHelper
/System/Library/PrivateFrameworks/TCC.framework/Versions/A/Resources/tccd
```

# systemextensionsctl
---

This is a small command line utility provided by Apple to "list and control System Extensions installed". It's also written in Objective-C. It makes direct use of the `OSSystemExtensionClient` class to call into `sysextd`. While this isn't pictured in the diagram above it's worth calling out because it's not well documented and is useful for anyone playing around with System Extensions. In the first Catalina beta it was actually non-functional but now if you run the command you can see some basic help information about it.

```shell
$ systemextensionctl
systemextensionsctl: usage:
    systemextensionsctl developer [on|off]
    systemextensionsctl list [category]
    systemextensionsctl reset  - reset all System Extensions state
    systemextensionsctl uninstall <teamId> <bundleId>; can also accept '-' for teamID
```

There are some other test commands not documented. They all basically allow further testing of calls into `sysextd`

```
    systemextensionsctl disable <teamID> <bundleID>; can also accept '-' or 'plat' for teamID
    systemextensionsctl enable <teamID> <bundleID>; can also accept '-' or 'plat' for teamID
    systemextensionsctl test <test_routine_name>
    systemextensionsctl testauthz <test_routine_name>
    systemextensionsctl testshouldmove <fromPath> <toPath>; n.b. paths not URLs. To test moving to trash, specify \\\"Trash\\\" as toPath.
    systemextensionsctl testwillmove <fromPath> <toPath>; n.b. paths not URLs. To test moving to trash, specify \\\"Trash\\\" as toPath.
```

Finally I think it's useful to see what entitlements applications like these have to get some idea of whether or not the functionality is locked down or not.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.private.system-extensions.modify</key>
    <true/>
</dict>
</plist>
```

# sysextd
---

This is a new system daemon. It's mostly written in swift, which is exciting to see. Most of the existing system daemons are written in Objective-C. It's great to see Apple adopting Swift more and more for internal uses other than just end user applications. This daemon is responsible for installation, approval, and uninstalling of all the system extensions. It has the following entitlements:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.private.security.storage.SystemExtensionManagement</key>
    <true/>
</dict>
</plist>
```

It delegates out actions to each of the relevant subsystems: `kextd`, `nesessionmanager` and `endpointsecurityd`. There are a couple internal classes that are worth calling out.

## KextdClient

This class forwards requests off to `kextd` through IOKit for DriverKit loading.

## StandardExtensionDelegate

This class provides common actions for forwarding requests off to any friend daemon that implements the `OSSystemExtensionPointListener` class.

## NetworkExtensionCategoryDelegate
## EndpointSecurityCategoryDelegate

These two classes leverage the `StandardExtensionDelegate` logic for the most part. They're both responsible for claiming their respective extension types.

# DriverKit
---

DriveKit extensions have a new `.dext` file type. There's a lot that goes on to make user space drivers work so again I won't go into it in great detail. The DriverKit extension binaries are a bit weird. They have no `LC_MAIN` command in the Mach-O binary. This means if you wanted to try to run the extension yourself in user space you couldn't. In fact if you try to run it in LLDB you'll get the following error: `error: Bad executable (or shared library)`. Ultimately the binaries do get loaded and run in user space. It would be interesting to dig a little further into the loading process of `.dext` files. Overall, I like the architecture and find it very interesting. The user mode drivers basically set up a connection to IOKit and get forwarded events happening and can send back responses as well.

## IOKit.framework

There's a couple new internal functions in the `IOKit.framework`.

### _KextManagerValidateExtension
### _KextManagerUpdateExtension
### _KextManagerStopExtension

These three functions are used by `kextd` to install and uninstall DriverKit extensions.

## kextd

There were changes to `kextd` in order for it to understand what drivers have a corresponding user executable. I'm hoping that Apple will release Catalina source shortly after final release in order to be able to look into this more.

## kernel

The kernel side of IOKit had changes to support sending events of to user space. It looks like in the kernel `IOUserServer` corresponds to the user space version of the `IOUserServer` class in the `DriverKit.framework`

# Endpoint Security
---

This is one of the most interesting  parts of System Extensions in my opinion. From what I can tell Apple took their standard pattern for kext to system daemon communication and attempted to turn it into a framework. That way third parties would only have to worry about writing the system daemon side of things and Apple could control the kernel side. Ideally this should lead to much better stability for third party security tools as they'll be kicked out of doing nasty things inside of kernel space.

## endpointsecurityd

This is another new system daemon. Written in Objective-C. It's responsible for the installing and uninstalling of endpoint security extensions. `libEndpointSecurity.dylib` doesn't communicate with `endpointsecurityd` in any way. There are a handful of entitlements the daemon has:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.private.endpoint-security.manager</key>
    <true/>
    <key>com.apple.private.security.storage.SystemExtensionManagement</key>
    <true/>
    <key>com.apple.private.system-extensions.extension-point</key>
    <true/>
    <key>com.apple.private.tcc.manager</key>
    <true/>
    <key>com.apple.private.xpc.protected-services</key>
    <true/>
</dict>
</plist>
```

## libEndpointSecurity.dylib

When you write an Endpoint Security extension this is the main library that you'll use. It makes use of the `IOKit.framework` to register itself as a client with the `EndpointSecurity.kext`. From there it can opt in or out of listening for certain events from the Kernel Extension and send responses back.

## EndpointSecurity.kext

If you've ever looked at any third party antivirus kernel extensions this kext looks very similar. It uses the MACF and kauth kernel frameworks to listen to events happening and then sends messages out to any registerd clients. Those registered clients can then make decision on whether to allow or deny the actions happening.

# Network Extensions
---

This is probably where I've spent the least amount of time looking into things. The `NetworkExtension.framework` was something that already existed. As part of Catalina some of the functionality that already existed on iOS is now ported over to macOS. With Network Extensions you can now create things like firewall applications that run completely in user space rather than having to use Network Kernel Extensions

## nesessionmanager

The daemon listens for messages from `sysextd` to take care of validating, starting and stopping network extensions

## NetworkExtension.framework

This framework existed but there is a new public function that should be noted.

### NEProvider.startSystemExtensionMode()

This method is responsible for starting up a listening XPC server. This allows the kernel to send out relevant messages to the user space Network Extensions.

# Conclusion
---

So as you can see there are a lot of different pieces involved in "System Extensions" and I only barely managed to scratch the surface. I'm excited to see how this evolves and the increased security it should bring.
