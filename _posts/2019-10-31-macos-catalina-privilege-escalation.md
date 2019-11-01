---
title: CVE-2019-8805 - A macOS Catalina privilege escalation
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

With the release of macOS Catalina in October, Apple rolled out a set of interesting new features collectively called [System Extensions](https://developer.apple.com/system-extensions/){: target="_blank"}. System Extensions are a set of user space frameworks encouraging developers who currently maintain and ship kernel extensions to move their features to user space for increased security and stability. One of these new frameworks is the [Endpoint Security](https://developer.apple.com/documentation/endpointsecurity){: target="_blank"} framework. As a security researcher this framework is of special interest. It’s intended to provide a public and stable API for implementing security products. During the process of looking into what functionality the Endpoint Security framework provided, a privilege escalation bug was identified that would let an attacker execute any code they wanted with root privileges. The following describes both the vulnerability as well as what Apple did to fix the issue.

# Endpoint Security Architecture

There are a lot of different parts to the Endpoint Security framework and in order to understand the vulnerability it will be useful to briefly cover the architecture of the Endpoint Security framework. The framework spans three different layers. There’s the user layer, the system layer, and then finally the kernel layer.

![Endpoint Security Architecture](/images/macos-catalina-privilege-escalation-1.png){: .align-center}

In order to use the Endpoint Security framework a developer needs to create at least two applications. One is an application that actually uses the Endpoint Security framework (`libEndpointSecurity.dylib`) and will run in the system layer. The other is an application running in the user layer that makes use of the System Extensions framework to actually prompt the user to approve and install the Endpoint Security extension. Apple takes care of creating and maintaining the components running in the kernel layer.

As can be seen in the diagram above, normally when loading an Endpoint Security extension the user application uses the `SystemExtensions.framework` and programmatically requests to install/load the specific extension. This sends a message to the `sysextd` daemon which is then responsible for sending a message to the `endpointsecurityd` daemon. The task that the `endpointsecurityd` daemon carries out is installing extensions and starting them up. To start them, the daemon calls into the `ServiceManagement.framework` calling `SMJobSubmit`. `SMJobSubmit` is what sends a message to `launchd` and it will launch the extension as a root daemon. Due to the way `launchd` works, even if a user exits the program, `launchd` will just restart it again.

# The Vulnerability

The privilege escalation vulnerability actually exists within `endpointsecurityd` and the `SystemExtensions.framework` it depends on. All of the communication above, between daemons, takes place using a low level system IPC mechanism called [XPC](https://developer.apple.com/documentation/xpc){: target="_blank"}. The `SystemExtensions.framework` provides a `OSSystemExtensionPointListener` class used by `endpointsecurityd` to listen for the XPC activation requests `sysextd` sends. When the `endpointsecurityd` daemon starts up it does the following:

![ESD Source](/images/macos-catalina-privilege-escalation-2.png){: .align-center}

So the `OSSystemExtensionPointListener` in the `SystemExtensions.framework` is what’s actually responsible for listening to XPC activation requests from `sysextd`.

XPC is a standard IPC mechanism on macOS that any application utilize and copmmunicate with. So what's stopping a malicious application from sending a XPC request to `endpointsecurityd` asking it to launch a malicious program? Normally the answer is entitlements. Apple’s built a good system in macOS and iOS where critical subsystems of the OS are protected by entitlement checks. A random binary has to have a special entitlement embedded in it and be properly signed to communicate with critical parts of the OS. In this case however the `OSSystemExtensionPointListener` class did not have any entitlement checks. 

This means that in macOS 10.15.0 an attacker can send a specially crafted XPC message directly to the `"com.apple.endpointsecurity.system-extensions"` service and request `launchd` to start any application the attacker chooses. Compoinding the issue, `launchd` will treat the application it’s starting as a system daemon. It will run it as root and ensure that the process is always running, even if the user tries to kill it.

# Apple's Patch

With the release of [macOS 10.15.1](https://support.apple.com/en-us/HT210722){: target="_blank"}, Apple has patched this vulnerability. If we disassemble and reconstruct the code for `[OSSystemExtensionPointListener listener:shouldAcceptNewConnection:]` we can see the changes that they applied:

![OSSystemExtensionPointListere diff](/images/macos-catalina-privilege-escalation-3.png){: .align-center}

The first thing the `[OSSystemExtensionPointListener listener:shouldAcceptNewConnection:]` method does now is call `valueForEntitlement` passing in the string `"com.apple.private.security.storage.SystemExtensionManagement"`. If the calling program does not have this entitlement then the message "XPC denied because caller lacks entitlement" will be logged and the connection will not be accepted. If you filter the macOS logs for `endpointsecurityd` and attempt to send the specially crafted XPC message you can see this error in the system logs:

```
default    16:04:04.919496-0400    endpointsecurityd Init ipc server
default    16:04:04.923851-0400    endpointsecurityd ESD init
default    16:04:04.926649-0400    endpointsecurityd XPC denied because caller lacks entitlement
```    

The only binary that has this special entitlement is `sysextd`. This means that now in macOS 10.15.1 only `sysextd` can ask `endpointsecurityd` to install and launch extensions.

# Conclusion

The new System Extensions framework in macOS Catalina is a great step forward for even more security and stability in macOS. In this case the vulnerability was fairly straightforward and Apple was able to patch it quickly. As Apple creates more and more system daemons to do specialized jobs however it does create an increased attack surface. In turn this creates more ways for attackers to find their way into the secure parts of the OS. As security researchers, this means we also need to be working to understand these new security mechanisms and testing them to see if they work as expected.
