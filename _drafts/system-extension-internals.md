---
title: System Extension internals
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

One of the most exciting things announced at this years WWDC [System Extensions](https://developer.apple.com/system-extensions/){: target="_blank"}. This is great from a security perspective. Apple has been attempting to wrangle in kexts for a while now and this seems to be a final nail in the coffin. They have said macOS 10.15 will be the last release to fully support kexts without compromises and that in future releases of macOS Kernel Extensions with System Extension equivalents will not load at all.

Disclaimer this is based on the Catalina beta so these details might change by the time of the final release

# SystemExtensions.framework

Written in Objective-C

[https://developer.apple.com/documentation/systemextensions](https://developer.apple.com/documentation/systemextensions){: target="_blank"}

Not much here. But what's not docuemnted?

### OSSystemExtensionInfo

This contains the full information pertaining to a system extension. This object conforms to `NSSecureCoding` and is used when making request to `OSSystemExtensionPointListener` instances.

### OSSystemExtensionPointListener

The main System Extension daemon has friend daemons that it uses to handle the different types of system extension categories. Both `endpointsecurityd`and `nesessionmanager` use this class at start up to create a XPC service that listens for system extension start and stop events.

### OSSystemExtensionClient

This class is the main way for the different applications to talk to `sysextd`. You can see a list of apps using this class by searching for it with ripgrep

```shell
$ rg -l -a -M 80 OSSystemExtensionClient 2> /dev/null
/sbin/kextload
/usr/sbin/kextcache
/usr/libexec/nesessionmanager
/usr/libexec/kextd
/usr/libexec/endpointsecurityd
/usr/bin/systemextensionsctl
/usr/bin/kextutil
```

# systemextensionsctl

Written in Objective-C/C. Uses the `OSSystemExtensionClient` class to call into `sysextd`

```shell
$ systemextensionctl
systemextensionsctl: usage:
	systemextensionsctl developer [on|off]
	systemextensionsctl list [category]
	systemextensionsctl reset  - reset all System Extensions state
	systemextensionsctl uninstall <teamId> <bundleId>; can also accept '-' for teamID
```

There are some other test commands not documented. The all basically allow further testing of calls into `sysextd`

```
	systemextensionsctl disable <teamID> <bundleID>; can also accept '-' or 'plat' for teamID
    systemextensionsctl enable <teamID> <bundleID>; can also accept '-' or 'plat' for teamID
    systemextensionsctl test <test_routine_name>
    systemextensionsctl testauthz <test_routine_name>
    systemextensionsctl testshouldmove <fromPath> <toPath>; n.b. paths not URLs. To test moving to trash, specify \\\"Trash\\\" as toPath.
    systemextensionsctl testwillmove <fromPath> <toPath>; n.b. paths not URLs. To test moving to trash, specify \\\"Trash\\\" as toPath.
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.private.system-extension.uninstall</key>
	<true/>
	<key>com.apple.private.system-extensions.manager</key>
	<true/>
</dict>
</plist>
```

# sysextd

New daemon. Mostly written in swift, which is exciting to see Apple use more for their system daemons.

This is really responsible for installation, approval, uninstalling of all the system extensions.

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

Delegates actions out to `kextd`, `nesessionmanager` and `endpointsecurityd`

## KextdClient

Talks to kextd through IOKit for DriverKit loading

## StubDelegateWHatever

Delgates out to different daemons that are registered as `OSSystemExtensionPointListener` classes

# DriverKit

dexts are weird

Mach-O 64-bit executable x86_64

lldb can't run it
error: Bad executable (or shared library)

Has no LC_MAIN

## IOKit.framework

### _KextManagerValidateExtension

### _KextManagerUpdateExtension

### _KextManagerStopExtension

## kextd

Change to understand has user executable

## kernel

IOKit changes in the kernel IOUserServer

# Network Extensions

This framework already existed. Moved over functionality that already existed on iOS.

## NetworkExtension.framework

### NEProvider.startSystemExtensionMode()

## nesessionmanager

The daemon listens for messages from `sysextd` to take care of validating, starting and stopping network extensions

# Endpoint Security

## endpointsecurityd

Not sure yet about the role played here. Because the EndpointSecurity.kext will send out calls to any subscribers using libEndpointSecurity.dylib

Responsible for loading installed endpoint security extensions?

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.private.endpoint-security.manager</key>
	<true/>
	<key>com.apple.private.system-extensions.extension-point</key>
	<true/>
	<key>com.apple.private.tcc.manager</key>
	<true/>
</dict>
</plist>
```

Really just for starting stopping and invalidating extensions

## EndpointSecurity.kext

Uses the MACF frameowrk to listen to things and sends them out to connected clients

## libEndpointSecurity.dylib

Uses IOKit to connect to the EndpointSecurity.kext as a client and register itself for notifications
