---
title: Virus scanning on macOS
categories:
  - Reverse Engineering
tags:
  - C
  - Kernel
  - macOS
---

Inspired by a recent post on the [MDSec blog](https://www.mdsec.co.uk/2018/08/endpoint-security-self-protection-on-macos/){: target="_blank"}, I wanted to take a deeper look at virus scanning on macOS. When I ran into issues getting McAfee kexts approved and loaded correctly it gave me a good excuse to dig deeper. In this post I will provide a brief overview of the different Kernel APIs available to virus scanning tools on macOS and specifically how McAffee's implementation works. Finally I will cover some things that I think could be improved in the McAfee implementation.

I'd like to include a similar disclaimer as the MDSec blog post. Root level permissions are needed to connect and interact with the McAfee implementation. If an attacker has root level permissions, then regardless of specific security product, the attacker is going to have fairly wide open access to the machine. That said, it's certainly worthwhile to understand how the security products we choose protect us and any issues there might be with them.

# Kernel APIs

Any virus scanning solution on any platform is going to want to inspect files on the computer. Additionally most have the ability to block malicious files before they're opened. The section below is meant to provide a brief overview of the most relevant KPIs that macOS has and provide some additional links to more documentation.

## Kauth

From Apple's excellent [tech note on Kauth](https://developer.apple.com/library/archive/technotes/tn2127/_index.html){: target="_blank"}:

>Mac OS X 10.4 Tiger introduced a new kernel subsystem, Kernel Authorization or Kauth for short, for managing authorization within the kernel. The Kauth subsystem exports a kernel programming interface (KPI) that allows third party kernel developers to authorize actions within the kernel, modify authorization decisions, and extend the kernel's authorization landscape. It can also be used as a notification mechanism.

The tech note goes on to say that these KPIs can specifically be used for developing an anti-virus product. The Kauth KPIs allow you to register listeners for different `scopes` built into the macOS kernel. There are two different scopes that are most useful for virus scanning: the file operation scope and the vnode scope. The file operation scope unfortunately does not let you block access to files in real time, only receieve notifications about actions happening to them. The vnode scope is a little more complex but allows you to authorize or deny the different file operations. In either case you need to make sure your kext code is not slow otherwise all file operations for the system will be slowed down.

## MAC

Mandatory Access Control, or MAC for short, came from the FreeBSD TrustedBSD framework. Unlike Kauth which only provides a handful of scopes, the MAC interface provides very granular hooks into almost every major part of the kernel. Unfortunately this is not an official Apple approved KPI. This means that Apple uses it, but they don't document it and don't recommend that you use it. This KPI is how Apple themselves implement their sandboxing mechanism in the kernel. Most security products will make use of this unofficial KPI in some fashion since it is so powerful. To get an idea of all the different hook points just take a look at the [mac_policy.h](https://github.com/knightsc/darwin-xnu/blob/master/security/mac_policy.h#L6305){: target="_blank"} header in the kernel.

For a good overview of writing a kext that uses the MAC framework, head over to the [Objective-See blog](https://objective-see.com/blog.html#blogEntry9){: target="_blank"}.

## Kernel Control API

Most kernel extensions are going to need to communicate with user space applications as well. There are different ways that a kext can communicate with user space but I'll just touch on the [Kernel Control API](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/NKEConceptual/control/control.html){: target="_blank"} here. This KPI provides a socket-based communication mechanism for your kext. After your extension registers itself, user space applications can use socket APIs along with a new `PF_SYSTEM` domain and `SYSPROTO_CONTROL` protocol. This allows your user space software to set options in your kext as well as send and receive information back and forth.

# McAfee Endpoint Security for Mac

*The following analysis applies to McAfee Endpoint Security for Mac Version 10.2.3 (3096)*

As I mentioned at the start of this post, one of the things that prompted me to dig deeper into McAfee was having issue getting all the kexts approved and loaded. As of macOS 10.13 [users are required to approve any third party kernel extension](https://developer.apple.com/library/archive/technotes/tn2459/_index.html){: target="_blank"} before they can be loaded. To start off, I searched for all of the kexts installed by McAfee.

```sh
$ sudo find /Library/Application\ Support/McAfee -name *.kext -print
/Library/Application Support/McAfee/AntiMalware/AVKext.kext
/Library/Application Support/McAfee/StatefulFirewall/SFKext.kext
/Library/Application Support/McAfee/FMP/FileCore.kext
/Library/Application Support/McAfee/FMP/mfeaac.kext
/Library/Application Support/McAfee/FMP/FMPSysCore.kext
```

Since I wanted to specifically learn more about virus scanning on macOS I started by analyzing the `AVKext.kext`.

## AVKext.kext

When reversing kexts, I always start by inspecting their `Info.plist` file. This provides an overview of what KPIs it uses as well as what other kexts it depends on. In the case of `AVKext.kext` it had the following defined:

```xml
<key>CFBundleIdentifier</key>
<string>com.McAfee.AVKext</string>
<key>OSBundleLibraries</key>
<dict>
    <key>com.apple.kpi.bsd</key>
    <string>8.0.0</string>
    <key>com.apple.kpi.libkern</key>
    <string>8.0.0</string>
    <key>com.intelsecurity.FileCore</key>
    <string>1</string>
</dict>
```

This kext uses the bsd and libkern kpis as well as one of the other McAfee kexts. Since this kext does not use `IOKit`, it means it will have a start and stop function that run when the kext is loaded or unloaded. If you look for the `_start` symbol it references a pointer called `_realmain`. This points to the function the kernel calls when the kext is loaded. In this case it is `AVKext_start`.

```c
kern_return_t AVKext_start(kmod_info_t * ki, void *d)
{
    printf("AV Kext: Entered");
    gAVLogLevel = LOG_ERROR;
    
    gAVMallocTag = OSMalloc_Tagalloc("com.McAfee.AVKext", OSMT_DEFAULT);
    if (!gAVMallocTag) {
        if (gAVLogLevel >= LOG_ERROR) {
            printf("MFE_AV: ERROR\"Could not allocate memory for kext\"\n");
        }
        
        return KERN_FAILURE;
    }
    
    gAVLockGroup = lck_grp_alloc_init("com.McAfee.AVKext", NULL);
    if (!gAVLockGroup) {
        if (gAVMallocTag) {
            OSMalloc_Tagfree(gAVMallocTag);
            gAVMallocTag = NULL;
        }
        return KERN_FAILURE;
    }
    
    if ((AVScan_init("com.McAfee.AVKext", 0x0) == 0x0) && 
        (AVScanBooster_init("com.McAfee.AVKext", 0x0) == 0x0)) {
        if (AV_userkernintf_init() == 0x0) {
            sysctl_register_oid(&sysctl__kern_com_mcafee_AV_log);
            gRegisteredOID = 0x1;
        }
    }

    return KERN_SUCCESS;
}
```

There's not a ton happening during startup. The kext creates a lock group and malloc tag. Then it initializes the `AVScan` and `AVScanBooster` subsystems which initialize some more locks and global data structures. Then it calls `AV_userkernintf_init` which sets up the kernel control api interface and finally it sets up a sysctl entry to control the log level. Using the commands below you can enable more detailed logging which is helpful while analyzing interactions with the kext.

```sh
$ sysctl -a | grep -i "mcafee\|intelsecurity"
kern.com_intelsecurity_filecore_log: 1
kern.com_intelsecurity_filecore_timeout: 5
kern.com_mcafee_AV_log: 2
kern.com_mcafee_syscore_log: 1
kern.com_mcafee_firewall_log: 1

$ sudo sysctl -w kern.com_mcafee_AV_log=5
kern.com_mcafee_AV_log: 2 -> 5

log stream --predicate 'processID == 0 && eventMessage CONTAINS[c] "MFE"'
```

The `HandleSet` method takes care of most of the orchestration between the user space scanner and the kext. There are many different commands that it understands but I will only cover some of the ones most relevant to basic virus scanning functionality. For more details on the `HandleSet` function please see the [reversed AVKext.c](https://gist.github.com/knightsc/4df9789ac96d428e1d5efb551d09074d){: target="_blank"} file.

### PING_KEXT

This command calls the `AV_ping_kext` function. It calls the `microuptime` function passing in a global `gWatchDogTime` variable. Even though the command doesn't do a lot, it's important to call out because other functions expect that the scanner is sending this command approximately once a minute to indicate that it's still alive and functioning.

### ENABLE_KERNEL_HOOK

This command calls the `AV_hook_register` function. The important thing to note is that this calls into the `Mfe_registerKextClient` function which is part of the `FileCore.kext`. The `FileCore.kext` is the code actually responsible for using the Kernel APIs mentioned above like Kauth and MAC. Additionally one of the parameters to the `Mfe_registerKextClient` function is `AV_Handle_Auth_Events`. This handle function is responsible for responding to file operations that `FileCore.kext` sends to `AVKext.kext`. Additionally, `AV_Handle_Auth_Events` is what actually calls `ctl_enqueuedata` which sends data over the established kernel control socket to the user space scanning application. The data sent to user space scanner takes the following format:

```c
struct FileScanMessage {
    int64_t inode;
    int32_t pid;
    int32_t uid;
    int32_t gid;
    int32_t devid;
    int32_t result;
    int32_t action;
    int32_t u1;
    int32_t u2;
    int64_t u3;
    char file[1024];
    int64_t u4;
};
```

The user space scanner can then inspect the file to make a decision about what to do. It then marks the action as approved or denied by setting a 1 or 2 respectively in the `result` field.

### PUT_SCAN_RESULT_MSG

When the scanner wants to send it's response back to the `AVKext.kext` it uses this command. It calls into the `AV_put_scan_result` function directly passing in the `data` and `len` arguements from the `HandleSet` function. This command is capable of hanlding multiple `FileScanMessage` records at a time. The size of the `FileScanMessage` is 1080 bytes. This command expects a buffer of at least 1088 bytes. The first 8 bytes are a count variable indicating how many `FileScanMessage` objects are included in the `data` buffer. `AV_put_scan_result` will cache some information about the message and then calls into `Mfe_handleFileEventResult` in the `FileCore.kext`.

## FileCore.kext

Again I started by inspecting the `Info.plist` for this kext. It had the following information:

```xml
<key>CFBundleIdentifier</key>
<string>com.intelsecurity.FileCore </string>
<key>OSBundleLibraries</key>
<dict>
    <key>com.apple.kpi.bsd</key>
    <string>12.0.0</string>
    <key>com.apple.kpi.dsep</key>
    <string>12.4.0</string>
    <key>com.apple.kpi.libkern</key>
    <string>12.0.0</string>
    <key>com.apple.kpi.mach</key>
    <string>14.0</string>
</dict>
```

Again, this kext does not use a ton of KPIs. Additionally it does not depend on any other kexts and does not use `IOKit`. Reviewing the start function for the kext reveals that it also sets up a kernel control interface as well as a sysctl entry for log level. 

### Mfe_registerKextClient / HandleConnect

Whether you connect to the `FileCore.kext` extension using the kernel control api or directly from another kext both of these functions do basically the same thing. They add a new entry onto a list of clients and increment the global `gNumClients` variable. Then they call the `registerForFileIOEvents` function. This function is what's responsible for hooking all of the file operations. It does the following three things:

* Calls `mac_policy_register` passing in a policy_conf configured to hook `mpo_vnode_check_rename`
* Calls `kauth_listen_scope` passing in the `com.apple.kauth.vnode` scope
* Calls `kauth_listen_scope` again passing in the `com.apple.kauth.fileop` scope

### Mfe_handleFileEventResult

This is the function called from the `PUT_SCAN_RESULT_MSG` command in `AVKext.kext`. This just calls into `handleFileEventResult` and the primary purpose of that is to call `wakeup` on the thread that was waiting on a response about a file.

## VShieldScanManager

I did't spend a lot of time looking into `VShieldScanManager`, but it can be found in the `/usr/local/McAfee/AntiMalware/` directory. I looked at this executable only to confirm the communication back and forth between `AVKext.kext` and the scanner. I created a proof of concept program to demonstrate the functionality of a basic scanner. It will connect to `AVKext.kext` and deny access to any files with a known word in the file path. You can see the full code here:

[https://gist.github.com/knightsc/f5417a577c14125426146be4d9a3864e](https://gist.github.com/knightsc/f5417a577c14125426146be4d9a3864e){: target="_blank"}

## Overall Architecture

Zooming out a little bit the overall architecture of the McAfee scanning solution looks something like this:

![McAfee Virus Scanning Architecture](/images/virus-scanning-on-macos-1.png){: .align-center width="75%"}

`VShieldScanManager` is launched from a LaunchDaemon and is responsible for loading and connecting to `AVKext.kext`. `AVKext.kext` then uses the `FileCore.kext` to subscribe to file operations. `FileCore.kext` is what actually uses the Kauth and MAC kernel APIs to actually hook into file operations. When a `FileScanMessage` is sent to the `VShieldScanManager` it will ask a `VShieldScanner` process to inspect and make a decision.

# What could be improved?

The following is a short list of things that can be improved in the kexts and installation of the McAfee product.

## 1. Verify the connecting scanning process

Ideally you only want your own software connecting and configuring your kext. Apple makes use of entitlement checks to do things like this but unfortunately they do not provide any standard way for kext developers to do the same. I have seen some kexts take the approach of requiring connecting code to pass in an authorization token. While this doesn't provide any real security, since it's easily reversed, it does raise the bar a little. By default macOS will only run software that has been code signed. So a better solution would be for the kext to validate the client based on it's code signature. The kext could use something like `csblob_get_teamid` to check and validate that the process connecting is a trusted one. In the case of McAfee, the `VShieldScanManager` is signed and they could validate that it's signed for `Team ID: GT8P3H7SPW` before allowing the process to connect.

## 2. Handle multiple connections properly

I mentioned earlier with the `FileCore.kext` that it handles multiple connections and keeps track of them all. Unfortunately the `AVKext.kext` does not. Take a look at the respective connect and disconnect functions for the kernel control interface.

```c
errno_t
HandleConnect(kern_ctl_ref ctlref, struct sockaddr_ctl *sac, void **unitinfo)
{
    if (gUnit == 0) {
        gUnit = sac->sc_unit;
        if (gAVLogLevel >= LOG_DEBUG) {
            printf("MFE_AV: DEBUG\"Scanner connected to kernel module\"\n");
        }
        return KERN_SUCCESS;
    } else {
        return ECONNREFUSED;
    }
}

errno_t
HandleDisconnect(kern_ctl_ref ctlref, unsigned int unit, void *unitinfo)
{
    if (gAVLogLevel >= LOG_DEBUG) {
        printf("MFE_AV: DEBUG\"Scanner disconnected from kernel module\"\n");
    }
    gUnit = 0;
    AV_hook_deregister();
    return KERN_SUCCESS;
}
```

This code is only expecting a single connection. When the connection is made it pulls out the `sc_unit` value and puts it into a `gUnit` global variable. On disconnect it resets this variable and then deregisters the hooks in `FileCore.kext`.

What this means is when `VShieldScanManager` is connected, if a second process tries to connect and disconnect, the `HandleDisconnect` function will blindly reset `gUnit` and deregister the hooks. If the second process then tries to connect again it will be successful since `gUnit` was set back to zero. You can try this for yourself using the [ScanManager.c](https://gist.github.com/knightsc/f5417a577c14125426146be4d9a3864e){: target="_blank"} sample code. (Reminder it has to be run as root). If you run it once on a machine with McAfee it will fail. If you run it a second time it will connect and you will start receiving the stream of file events on the command line. What's worse, is since the scanner connection has been hijacked, you will no longer receive real time virus notification from the McAfee user interface. Also since `VShieldScanManager` is still technically running (just not connected) the user interface thinks everything is fine and threat prevention is operating normally.

I think there's a simple fix here. When `HandleDisconnect` is called, before doing anything else, the value of the passed in `unit` variable should be compared with the `gUnit` global variable. If it doesn't match, then this isn't the currently connected scanner and an error code should be returned. This won't completely stop the ability to hijack the scanning process but it would make it harder. If this change were made you would need to first kill the scanning process and then try to connect before it does.

## 3. Sanitize the kext output

Any kext that passes information back to user space should be careful to sanitize the data it sends back. As anyone who follows the iOS jailbreak scene knows, a leaked kernel pointer can help calculate where the kernel is loaded in memory and make further explotation possible. Take a look at a sample of the data sent from the `AVKext.kext` to the scanner process below.

```
inode:    0x00000000003c950e
pid:      444
uid:      502
gid:      20
devid:    0x01000004
result:   0x00000001
action:   0x00000064
unknown1: 0x00000001
unknown2: 0x00000001
unknown3: 0xffffff8000000000
file:     /Users/user1/Downloads/Clapzok/Clapzok
unknown4: 0xffffff802a8d8550
```

This data looks fairly straight forward but remember the `file` variable is defined as `char file[1024]`. So if we inspect the full set of data sent back from the kext to user space we'll see the following:

<script src="https://gist.github.com/knightsc/8748378829c54a2c528f4d1e104f1ea8.js"></script>

There's all sort of extra data in the `file` array after the actual end of the string. A lot of the data looks to be kernel memory addresses. `00 10 D4 20 80 FF FF FF` for example, would be the 64 bit number `0xffffff8929d41000`. This looks like a structure that was initially allocated on the stack in the kernel, some fields were set and then it was sent back over to user space.

I think the fix here is easy as well. The data structures sent back to user space should be properly initialized. If this structure was zeroed out before use we wouldn't have any of the extra garbage that we see above.

## 4. Sanitize the kext input

In addition to sanitizing the data sent to user space, any kext should be extra careful to sanitize data coming in from user space. When `AVKext.kext` starts up and calls the `ctl_register` function it sets the `ctl_sendsize` and `ctl_recvsize` large enough to hold up to a thousand `FileScanMessage` instances. When the `PUT_SCAN_RESULT_MSG` message is sent the first field is a count. It does not look like any bounds checking is done on the count sent in. If you set the count to `0xffffffffffffffff` the result is a kernel panic with the following stack trace:

```
(lldb) bt
* thread #2, name = '0xffffff80200d3530', queue = '0x0', stop reason = signal SIGSTOP
  * frame #0: 0xffffff800f1a405b kernel.development`memcpy + 11
    frame #1: 0xffffff7f918e6d84 AVKext`AV_put_scan_result + 340
    frame #2: 0xffffff7f918e75ea AVKext`HandleSet + 202
    frame #3: 0xffffff800f804378 kernel.development`ctl_ctloutput(so=0x00000000000122c3, sopt=0xffffff801b9db680) at kern_control.c:1206 [opt]
    frame #4: 0xffffff800f888137 kernel.development`sosetoptlock(so=0xffffff8020bb26b0, sopt=0xffffff90aadcbee8, dolock=1) at uipc_socket.c:4802 [opt]
    frame #5: 0xffffff800f89712e kernel.development`setsockopt(p=<unavailable>, uap=0xffffff802012a000, retval=<unavailable>) at uipc_syscalls.c:2421 [opt]
    frame #6: 0xffffff800f958358 kernel.development`unix_syscall64(state=0xffffff801c1bb140) at systemcalls.c:381 [opt]
    frame #7: 0xffffff800f223466 kernel.development`hndl_unix_scall64 + 22
```

The `AV_put_scan_result` function blindly tries to read past the end of the data and the result is a crash. I don't believe that this crash can lead to full kernel control but there's no reason to allow this to happen. Since there seems to be an implied limit passed on the `ctl_register` call it makes sense to fix this issue by adding a simple bounds check to the receieved count.

## 5. Improve self protections

McAfee Endpoint Security alows administrators to set an ePO administrator password. Users get asked for this password before being allowed to modify preferences. If a local user has root access though it seems like most components can be directly controlled from the command line.

For example if you want to disable the real time notifications you do not need to hijack the scanner you can simply execute the following commands:

```sh
$ ps xa | grep VShieldScan | grep -v grep
 1064   ??  Ss     0:03.98 /usr/local/McAfee/AntiMalware/VShieldScanner
 1067   ??  Ss     0:00.05 /usr/local/McAfee/AntiMalware/VShieldScanManager
 1089   ??  S      0:00.00 /usr/local/McAfee/AntiMalware/VShieldScanner
 1090   ??  S      0:00.00 /usr/local/McAfee/AntiMalware/VShieldScanner
 1091   ??  S      0:00.00 /usr/local/McAfee/AntiMalware/VShieldScanner

$ sudo /usr/local/McAfee/AntiMalware/VSControl stopoas
Stopping OAS...

$ ps xa | grep VShieldScan | grep -v grep
$
```

This would seem to defeat the purpose of having the ability to set an ePO administrator password required for preference changes. Additionally, at least in my current setup, a root user can completely uninstall the software even if they don't know the ePO administrator password. Now this might be intended functionality, but overall I didn't see much in the way of self protection features. Adding improved self protection features would probably be a larger change but might really be worth it.

# Conclusion

Overall, I didn't find any major issues with the McAfee implementation. I do think the suggesstions above would help improve the reliability and stability of it though. Similar to how Apple has used SIP to lock down the damage a root level user can do it makes sense that kext developers should try to take the same approach. I'd love to see Apple help provide improved KPIs to make the process of securing kexts even easier.
