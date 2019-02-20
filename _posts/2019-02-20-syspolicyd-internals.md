---
title: syspolicyd internals
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

With my [previous post]({% post_url 2019-01-23-system-policy %}){: target="_blank"} I took a look at the `SystemPolicy.framework` and how it kept track of 32-bit applications that had been run. In the process of looking into that I ended up looking into the internals of `syspolicyd`. Way back in macOS 10.10.5 `syspolicyd` was part of the [security_systemkeychain](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55202/syspolicyd/syspolicyd.cpp.auto.html){: target="_blank"} source code that Apple releases with each version of macOS. Unfortunately since that time `syspolicyd` was moved out of the security_systemkeychain package and closed sourced. This post details the internals of `syspolicyd` as it is today in macOS 10.14.x and covers both what services it provides and what clients connect and use its functionality.

# Overview

`syspolicyd` was originally introduced in macOS 10.7.3 with the Gatekeeper feature. Its original purpose was to act as the centralized daemon for answering Gatekeeper questions. Today it still serves that purpose but its scope has greatly expanded. In addition to assessing applications before running, the daemon also handles authorizing the loading of KEXTs as well as tracking legacy applications that the user has run. In Mojave `syspolicyd` has expanded again and is responsible for handling software notarization checks as well. We'll start with a very high level look at the daemon startup process and then dive deeper into each of `syspolicyd`'s subsystems.  

# Startup

The startup process of the `syspolicyd` daemon is not very complex. The entire main function was outlined in my [previous article]({% post_url 2019-01-23-system-policy %}){: target="_blank"}. Here it is in its entirety:

```c
int main() {
    KextManagerService *kextManagerService = [[KextManagerService alloc] init];
    [kextManagerService registerActivities];
    [kextManagerService startXPCService];
    
    ExecManagerService *execManagerService = [[ExecManagerService globalManager] retain];
    [execManagerService registerActivities];
    [execManagerService startMIGService];
    [execManagerService startXPCService];
    
    createSystemPolicyDatabase();
    registerSystemPolicyService();
    
    NSRunLoop *mainRunLoop = [[NSRunLoop mainRunLoop] retain];
    [mainRunLoop run];

    [mainRunLoop release];
    [execManagerService release];
    [kextManagerService release];

    return 0x1;
}
```

Conceptually you can think of `syspolicyd` as being composed on three main subsystems:

* [Gatekeeper](#gatekeeper)
* [KextManager](#kextmanager)
* [ExecManager](#execmanager)

When [launchd](#launchd) starts up `syspolicyd` each subsystem goes through and initializes itself. They set up database connections, XPC services and activities and even a MIG service. Each subsystem is detailed out in the sections below

# Gatekeeper

Gatekeeper is a technology that helps ensure you only run software from the app store and trusted developers. It's responsible for checking the code signing attributes on applications downloaded from the internet.

## Services

### com.apple.security.syspolicy

Based on the previously open sourced [syspolicyd.cpp](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55202/syspolicyd/syspolicyd.cpp.auto.html){: target="_blank"} code, we can easily get a sense of how this XPC service works. Essentially the service checks the incoming XPC message dictionary for what `function` is sent over. If the `function` sent over is known then the corresponding function implementation is called. Otherwise if the `function` isn't passed or is unknown then an error is returned to the XPC client. The following `function` values are currently defined as of macOS Mojave:

* [assess](#assess)
* [update](#update)
* [record](#record)
* [cancel](#cancel)
* [check-dev-id](#check-dev-id)
* [check-notarized](#check-notarized)
* [ticket-register](#ticket-register)
* [ticket-lookup](#ticket-lookup)

Most of these functions call into the `Security.framework` to do the real work. Which can be somewhat confusing since we'll see later that the `Security.framework` is one of the primary clients of this service. The framework is set up in such a way that when regular processes call into its APIs it will send an XPC request to this service and when this service runs it will run the `PolicyEngine` directly inside of `syspolicyd` using the frameworks code.

#### assess

Calls into `SecAssessmentCreate` passing in the `kSecAssessmentFlagDirect` flag so that the policy engine runs in process. The `PolicyEngine::evaluate` method is then called. This method asks the system for its assessment of a proposed operation. A CFURL is passed in describing the file central to the operation. The operation defaults to assessing whether the file can be executed or not but there are also options for evaluating if software can be installed or a document can be opened.

#### update

Calls into `SecAssessmentCopyUpdate` which will then in turn call into `PolicyEngine::update`. This method is responsible for making changes to the system policy configuration. The configuration gets stored in a sqlite3 database in `/var/db/SystemPolicy`.

#### record

This method is called into through `SecAssessmentControl` with the operation value being passed in as `ui-record-reject-local`. This method will do a check to ensure the caller holds the `com.apple.private.assessment.recording` entitlement before calling into `PolicyEngine::recordFailure`. This writes the failure out to the `/var/db/.LastGKReject` file.

#### cancel

Cancels an in progress assessment request.

#### check-dev-id

This method will check whether Gatekeeper allows apps from identified developers. `SecAssessmentControl` is called with the operation `ui-get-devid-local` passed in. This will in turn run the following query on the `/var/db/SystemPolicy` database.

```sql
SELECT disabled
FROM authority
WHERE label = 'Developer ID';
```

Even though there can be multiple records returned from this query, the `PolicyEngine` simply returns false if the value of disabled is 1 otherwise it returns true.

#### check-notarized

This is similar to `check-dev-id`. It will check whether Gatekeeper allows notarized applications to run. This again calls `SecAssessmentControl` this time passing in the `ui-get-notarized-local` option. This in turn runs the following query:

```sql
SELECT disabled
FROM authority
WHERE label = 'Notarized Developer ID';
```

If disabled is 1 then the `PolicyEngine` returns false otherwise it returns true.

#### ticket-register

When the system is validating binaries it will attempt to register stapled notarization tickets it finds with the system. This service function ultimately handles those calls. The data passed in gets turned into a `GKTicket` object and then validated. If the ticket is validated then the following query is run to insert the information into the `/var/db/SystemPolicyConfiguration/Tickets` sqlite3 database.

```sql
INSERT INTO tickets (hash, hash_type, timestamp, flags)
VALUES (?1, ?2, ?3, ?4)
```

#### ticket-lookup

This function is responsible for retrieving ticket information. It can either retrieve the information from the locally stored copy in `/var/db/SystemPolicyConfiguration/Tickets` or it can query a web service at Apple to lookup that information. In some cases when the `Security.framework` wants to check if a ticket is revoked it will force the lookup in the web service provided by Apple. Here's an example of the query checking for a local copy of the ticket:

```sql
SELECT tickets.flags
FROM hashes INNER JOIN tickets ON hashes.ticket_id = tickets.id
WHERE hashes.hash = ?1 AND hashes.hash_type = ?2
```

And here's an example of a lookup of a ticket in the web service:

```shell
$ curl --data '{"records":[{"recordName":"2/2/4dca04a3465b95866423323d7f3e1e31ad3ac0ef"}]}' \
https://api.apple-cloudkit.com/database/1/com.apple.gk.ticket-delivery/production/public/records/lookup
{
  "records" : [ {
    "recordName" : "2/2/4dca04a3465b95866423323d7f3e1e31ad3ac0ef",
    "recordType" : "DeveloperIDTicket",
    "fields" : {
      "signedTicket" : {
        "value" : "czhjaAEAAADwBQAA/wAAADCCBewwggL+MIICpKADAgECAggcrXLgBzKYBDAKBggqhkjOPQQDAjByMSYwJAYDVQQDDB1BcHBsZSBTeXN0ZW0gSW50ZWdyYXRpb24gQ0EgNDEmMCQGA1UECwwdQXBwbGUgQ2VydGlmaWNhdGlvbiBBdXRob3JpdHkxEzARBgNVBAoMCkFwcGxlIEluYy4xCzAJBgNVBAYTAlVTMB4XDTE4MDUwMzA1MjI1N1oXDTE5MDYwMjA1MjI1N1owRDEgMB4GA1UEAwwXU29mdHdhcmUgVGlja2V0IFNpZ25pbmcxEzARBgNVBAoMCkFwcGxlIEluYy4xCzAJBgNVBAYTAlVTMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEJbqyhMrgDDTnHBoZheGp0mypFXTwAUJKmXKQamgz95BKOEzvSlkeBxp1oI7mMSewrQLbOjztegUxnaB4RAtOAqOCAVAwggFMMAwGA1UdEwEB/wQCMAAwHwYDVR0jBBgwFoAUeke6OIoVJEgiRs2+jxokezQDKmkwQQYIKwYBBQUHAQEENTAzMDEGCCsGAQUFBzABhiVodHRwOi8vb2NzcC5hcHBsZS5jb20vb2NzcDAzLWFzaWNhNDAyMIGWBgNVHSAEgY4wgYswgYgGCSqGSIb3Y2QFATB7MHkGCCsGAQUFBwICMG0Ma1RoaXMgY2VydGlmaWNhdGUgaXMgdG8gYmUgdXNlZCBleGNsdXNpdmVseSBmb3IgZnVuY3Rpb25zIGludGVybmFsIHRvIEFwcGxlIFByb2R1Y3RzIGFuZC9vciBBcHBsZSBwcm9jZXNzZXMuMB0GA1UdDgQWBBSvkaFMaSOwRfsIVbF12z5+m2d5XjAOBgNVHQ8BAf8EBAMCB4AwEAYKKoZIhvdjZAYBHgQCBQAwCgYIKoZIzj0EAwIDSAAwRQIgWUBuPT4qbzW2paWYyyLINmhuQphzZj8ZXnNuflZB5kECIQCZBAp3F09H5C4WdMQdX3RUPL1a6udCdIi7QWtRMsiuOjCCAuYwggJtoAMCAQICCDMN7vi/TGguMAoGCCqGSM49BAMDMGcxGzAZBgNVBAMMEkFwcGxlIFJvb3QgQ0EgLSBHMzEmMCQGA1UECwwdQXBwbGUgQ2VydGlmaWNhdGlvbiBBdXRob3JpdHkxEzARBgNVBAoMCkFwcGxlIEluYy4xCzAJBgNVBAYTAlVTMB4XDTE3MDIyMjIyMjMyMloXDTMyMDIxODAwMDAwMFowcjEmMCQGA1UEAwwdQXBwbGUgU3lzdGVtIEludGVncmF0aW9uIENBIDQxJjAkBgNVBAsMHUFwcGxlIENlcnRpZmljYXRpb24gQXV0aG9yaXR5MRMwEQYDVQQKDApBcHBsZSBJbmMuMQswCQYDVQQGEwJVUzBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABAZrpFZvfZ8n0c42jpIbVs1UNmRKyZRomfrJIH7i9VgP3OJq6xlHLy7vO6QBtAETRHxaJq2gnCkliuXmBm9PfFqjgfcwgfQwDwYDVR0TAQH/BAUwAwEB/zAfBgNVHSMEGDAWgBS7sN6hWDOImqSKmd6+veuv2sskqzBGBggrBgEFBQcBAQQ6MDgwNgYIKwYBBQUHMAGGKmh0dHA6Ly9vY3NwLmFwcGxlLmNvbS9vY3NwMDMtYXBwbGVyb290Y2FnMzA3BgNVHR8EMDAuMCygKqAohiZodHRwOi8vY3JsLmFwcGxlLmNvbS9hcHBsZXJvb3RjYWczLmNybDAdBgNVHQ4EFgQUeke6OIoVJEgiRs2+jxokezQDKmkwDgYDVR0PAQH/BAQDAgEGMBAGCiqGSIb3Y2QGAhEEAgUAMAoGCCqGSM49BAMDA2cAMGQCMBUMqY7Gr5Zpa6ef3VzUA1lsrlLUYMaLduC3xaLxCXzgmuNrseN8McQneqeOif2rdwIwYTMg8Sn/+YcyrinIZD12e1Gk0gIvdr5gIpHx1Tp13LTixiqW/sYJ3EpP1STw/MqyZzh0awIAFAALAAAAAAAAAJd93lsAAAAAAk3KBKNGW5WGZCMyPX8+HjGtOsDvAlDgR63nlb6pEA+hkKhE0q0tT4RQAguPF5TMKygcsuMhhi36bua9337yAjOffZ5ht4cOrdIBiX3SixrrdpsKAlUlAxz4ULfGd9TMxRWNHjRrvfJKAh3TWXvQ6+MRu2uJj5ox3gLFyCbbAomqveGyGUoCClkGwyDZN75QVqH0AkA3ZKSK+Z1j8DpYPVCf/lo0/EloAkx/LZhS933RUvLZ+VBrlY5AxFcdAjKDMyQKcVLskN7lslJ9Vl898FKaAiXYL/9FfhqeZNILlAFiM6JIl+C+MEYCIQC7zJ0aWeNQ44IOuTv8A33kA40oOp997lqGNrMuS5EyQwIhANjfZz5W32jo5NauOHtAQkr21mB1kD3ztTbSSNSSU69w",
        "type" : "BYTES"
      }
    },
    "pluginFields" : { },
    "recordChangeTag" : "jnz4hhh6",
    "created" : {
      "timestamp" : 1541108961982,
      "userRecordName" : "_d28c74d190a3782e89496b0a13437fef",
      "deviceID" : "2"
    },
    "modified" : {
      "timestamp" : 1541307799566,
      "userRecordName" : "_d28c74d190a3782e89496b0a13437fef",
      "deviceID" : "2"
    },
    "deleted" : false
  } ]
}
```

## Clients

### Security.framework

The `Security.framework` is the primary client of this service with all of its SecAssessment APIs. As mentioned above if you call into these APIs outside of `syspolicyd` then the framework will use XPC to request the information from `syspolicyd`. Here's a short overview of the SecAssessment APIs:

| API                         | Description                                                |
|-----------------------------|------------------------------------------------------------|
| SecAssessmentCreate         | Ask the system for its assessment of a proposed operation. |
| SecAssessmentCopyUpdate     | Make changes to the system policy configuration.           |
| SecAssessmentControl        | Miscellaneous system policy operations.                    |
| SecAssessmentTicketRegister | Registering stapled ticket with system.                    |
| SecAssessmentTicketLookup   | Check whether software is notarized.                       |

Luckily the `Security.framework` is still open sourced, so you can look more into the internals of these APIs here:

[https://opensource.apple.com/tarballs/Security/Security-58286.220.15.tar.gz](https://opensource.apple.com/tarballs/Security/Security-58286.220.15.tar.gz){: target="_blank"}

Additionally if you're looking for a short sample of how the `Security.framework` makes the XPC calls to call into the Gatekeeper XPC service, I have some short example code [here](https://gist.github.com/knightsc/9b3fba827317cbf980ad8e99d8a17c7f){: target="_blank"}.

This means that if we really want to find out what clients use this service we should search the system for users of the SecAssessment APIs. I used ripgrep on a Mojave system to search for binaries that used `SecAssessment` and found the following items:

* /Applications/Utilities/Script Editor.app/Contents/MacOS/Script Editor
* /System/Library/CoreServices/CoreServicesUIAgent.app/Contents/MacOS/CoreServicesUIAgent
* /System/Library/CoreServices/Installer.app/Contents/MacOS/Installer
* /System/Library/Frameworks/Automator.framework/Versions/A/Automator
* /System/Library/Frameworks/JavaVM.framework/Versions/A/Frameworks/JavaRuntimeSupport.framework/Versions/A/JavaRuntimeSupport
* /System/Library/Frameworks/Security.framework/Versions/A/Security
* /System/Library/PreferencePanes/Security.prefPane/Contents/MacOS/Security
* /System/Library/PrivateFrameworks/ConfigurationProfiles.framework/XPCServices/SystemPolicyService.xpc/Contents/MacOS/SystemPolicyService
* /System/Library/PrivateFrameworks/PackageKit.framework/Versions/A/PackageKit
* /System/Library/PrivateFrameworks/SystemAdministration.framework/XPCServices/writeconfig.xpc/Contents/MacOS/writeconfig
* /System/Library/PrivateFrameworks/XprotectFramework.framework/Versions/A/XPCServices/XprotectService.xpc/Contents/MacOS/XprotectService
* /usr/libexec/mdmclient
* /usr/libexec/syspolicyd
* /usr/sbin/spctl

# KextManager

The KextManager subsystem plays a role in [user-approved kernel extension loading](https://developer.apple.com/library/archive/technotes/tn2459/_index.html){: target="_blank"}. The following Objective-C classes make up the core of the KextManager subsystem:

* KextManagerService
* KextManagerPolicy
* KextManagerDatabase
* KextManagerUI

The diagram below shows the flow of a typical request to load a kernel extension all the way to the point of notifying the user about it.

![KextManager Subsystem](/images/syspolicyd-internals-1.png){: .align-center}

## Services

### com.apple.security.syspolicy.kext

This XPC service is defined in the `syspolicyd` launchd plist. It's the primary way clients can check on and update what kernel extensions have been approved to load. Since it's written in Objective-C it makes use of the `NSXPCInterface` and `NSXPCListener` classes. The `NSXPCInterface` is defined to implement the following protocol:

```c
@protocol KernelExtensionManager
- (void)copyPendingApprovalsWithReply:(void (^)(NSArray *, NSError *))arg1;
- (void)copyCurrentPolicyWithReply:(void (^)(NSArray *, NSError *))arg1;
- (void)removeMDMPayload:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
- (void)installMDMPayload:(NSString *)arg1 withTeams:(NSArray *)arg2 andExtensions:(NSDictionary *)arg3 withReply:(void (^)(BOOL, NSError *))arg4;
- (void)setUserApprovalAllowed:(BOOL)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
- (void)updatePolicyItems:(NSArray *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
- (void)teamIdentifierIsAllowed:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
- (void)canLoadKernelExtensionInCache:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
- (void)canLoadKernelExtension:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
@end
```

The `KextManagerService` class is then responsible for implementing this protocol. Since KEXT loading is a sensitive system operation there are a handful of different permissions and entitlements needed to call into this XPC service. I'll break down my description of the services functionality by the entitlements needed to call into the different methods.

#### No entitlements needed

* `copyPendingApprovalsWithReply:`  
This method will retrieve the list of KEXTs that attempted to load but have not been approved yet from the `/var/db/SystemPolicyConfiguration/KextPolicy` sqlite3 database.

* `copyCurrentPolicyWithReply:`  
Provides a list of all KEXTs stored in the `/var/db/SystemPolicyConfiguration/KextPolicy` database.

* `teamIdentifierIsAllowed:withReply:`  
Determines if a specific team identifier is allowed by the `/var/db/SystemPolicyConfiguration/KextPolicy` database.

#### com.apple.rootless.kext-secure-management

* `canLoadKernelExtensionInCache:withReply:`  
* `canLoadKernelExtension:withReply:`  
Both of these methods end up doing the same thing. They call into the `KextManagerPolicy` object calling the `canLoadKernelExtensionAtURL:isCacheLoad:` selector. These are the main methods responsible for determining if a KEXT can be loaded. It will check if the KEXT is allowed by MDM or approved by the user.

#### com.apple.private.security.syspolicy.kext-policy-management

* `updatePolicyItems:withReply:`  
This method gets used when a KEXT is actually approved to update the record in the `/var/db/SystemPolicyConfiguration/KextPolicy` database.

#### com.apple.private.security.syspolicy.mdm-kext-policy-management

* `setUserApprovalAllowed:withReply:`  
Allows MDM to determine whether a user can approve a KEXT from the System Preferences or only the MDM whitelist is allowed.

* `installMDMPayload:withTeams:andExtensions:withReply:`  
Parses the MDM KEXT policy payload and updates the `kext_policy_mdm` table in the `/var/db/SystemPolicyConfiguration/KextPolicy` database.

* `removeMDMPayload:withReply:`  
Removes an existing MDM KEXT Policy from the `/var/db/SystemPolicyConfiguration/KextPolicy` database.

## Clients

If you attempt to find clients of this service by searching for users of the `KernelExtensionManager` protocol you're only going to find a single client, the `SystemPolicy.framework`. 

### SystemPolicy.framework

So in order to find the true clients we need to search the system for the binaries making use of the `SPKernelExtensionPolicy` class from the `SystemPolicy.framework`. Doing that results in the following list:

* /System/Library/PreferencePanes/Security.prefPane/Contents/MacOS/Security
* /System/Library/PrivateFrameworks/ConfigurationProfiles.framework/XPCServices/ExecutionPolicyService.xpc/Contents/MacOS/ExecutionPolicyService
* /System/Library/SystemProfiler/SPDisabledApplicationsReporter.spreporter/Contents/MacOS/SPDisabledApplicationsReporter
* /sbin/kextload
* /usr/bin/kextutil
* /usr/libexec/amfid
* /usr/libexec/kextd
* /usr/libexec/syspolicyd
* /usr/sbin/kextcache
* /usr/sbin/spctl

# ExecManager

As opposed to the `KextManager` subsystem the `ExecManager` subsystem actually has multiple services that it provides. One is an XPC service that has a similar architecture to the `KextManagerService` and the other is a MIG service that provides communication with the kernel. The core of the subsystem is made up of the following Objective-C classes:

* ExecManagerService
* ExecManagerPolicy
* ExecManagerDatabase

Unlike the `KextManagerPolicy` which has a helper class to notify the user through the UI, the `ExecManagerPolicy` class coordinates with the `CoreServices.framework` to notify the user of the launch of deprecated 32-bit executables.

## Services

### com.apple.security.AppleSystemPolicy.mig

Looking at the launchd plist for `syspolicyd` we see that this service is actually a `MachService`. Additionally it's defined to provide its service using `HostSpecialPort` 29. Looking at the `ExecManagerService` class we see that there's a method called `startMIGService` that is responsible for calling `bootstrap_check_in` to notify launchd the service is available. I created a short [Hopper](https://www.hopperapp.com/){: target="_blank"} script called [MIG Detect.py](https://github.com/knightsc/hopper/blob/master/scripts/MIG%20Detect.py){: target="_blank"} to find and label the different MIG messages that were being used.

```
0x10003ad70: MIG Subsystem 18600: 4 messages
0x10003ad98: MIG msg 18600
0x10003adc0: MIG msg 18601
0x10003ade8: MIG msg 18602
0x10003ae10: MIG msg 18603
```

#### notify_32bit_exec
#### notify_32bit_mmap

Both of these functions take similar actions. They call into the `ExecManagerService` and call the `notify32bitExecForUser` method. This will in turn save the information into the `/var/db/SystemPolicyConfiguration/ExecPolicy` sqlite3 database in the `legacy_exec_history_v4` table.
    
#### log_platform_binary

This method will call into the `ExecManagerService` and call the `logOldPlatformBinary` method. This will in turn create a `AWDSystemPolicyOldPlatformBinary` object and then call `AWDPostMetric` which is actually part of the private `WirelessDiagnostics.framework`. This private framework apparently collects more than just wireless diagnostics. Additionally a record gets added into the `old_platform_cache` table in the `/var/db/SystemPolicyConfiguration/ExecPolicy` database noting that the metric has been logged.

#### log_executable

This method will call into the `ExecManagerService` and call the `logExecutable` method. This eventually saves the information into the `scan_targets_v2` table in the `/var/db/SystemPolicyConfiguration/ExecPolicy` database. These records gets used from periodic XPC activities to measure performance metrics.

### com.apple.security.syspolicy.exec

This XPC service is defined in the `syspolicyd` launchd plist. Similar to the `KextManager` subsystem it's an Objective-C XPC service. The `NSXPCInterface` is defined to implement the following interface:

```c
@protocol ExecManager
- (void)copyLegacyExecutionHistoryWithReply:(void (^)(NSArray *, NSError *))arg1;
@end
```

There's not much to this XPC service. It simply provides a way for clients to retrieve a list of 32-bit applications that have been executed. No special entitlements are needed to call into this service.

## Clients

### AppleSystemPolicy.kext

As far as I could tell this is the only client of the `syspolicyd` MIG service. Since `syspolicyd` provides this as a host special port the kernel extension does the following when it's main `IOService` class is initialized:

```c
bool AppleSystemPolicy::init(OSDictionary *dict)
{
    kern_return_t kr;

    if (OSCompareAndSwapPtr(nullptr, this, &AppleSystemPolicy::_instance)) {
        // Initializes instance variables

        kr = host_get_special_port(host_priv_self(), HOST_LOCAL_NODE, HOST_SYSPOLICYD_PORT, &daemonPort);
        if (kr != KERN_SUCCESS) {
            os_log("Unable to retrieve syspolicyd port, failing: %d", kr);
            return false;
        }

        // Calls sysctl_register_oid to share performance stats with user space
        initPerformanceInfo();

        if (!super::init(dict)) {
            IOLog("Failed to initilize parent class");
            return false;
        }
        
        return true;
    }
    else {
        IOLog("AppleSystemPolicy already initialized!");
        return false;
    }
}
```

The kernel extension then has it's start method called which does the following:

```c
bool AppleSystemPolicy::start(IOService *provider)
{
    registerMACPolicy();
    if (!super::start(provider)) {
        os_log("Failed to start AppleSystemPolicy parent");
        return false;
    }
    else {
        os_log("AppleSystemPolicy has been successfully started");
        return true;
    }
}
```

The following are the MACF operations that get hooked:

* mpo_proc_notify_exec_complete
* mpo_file_check_library_validation
* mpo_file_check_mmap

All three of these hooks are simply used to inspect executables that are about to run or libraries that will be mapped into memory and check if they are 32-bit. If they are then a MIG call is made to inform `syspolicyd` of the information. If you want to see an example of what the MIG call itself looks like you can take a look at this small sample app I created called [aspmig.c](https://gist.github.com/knightsc/6aa75ec6dc8940fa7d381140fb57236a){: target="_blank"}. It manually calls the `notify_32bit_exec` MIG service.

If you want to look at the statistics of how long the MACF hooks take to run you can use the `sysctl` command line utility.

```shell
$ sysctl -a | grep asp
security.mac.asp.cache_entry_count: 3417
security.mac.asp.cache_allocation_count: 4104
security.mac.asp.cache_release_count: 687
security.mac.asp.exec_hook_time: 616061
security.mac.asp.exec_hook_count: 252162
security.mac.asp.library_hook_time: 2385933
security.mac.asp.library_hook_count: 1051821
```

### SystemPolicy.framework

Similar to the `KextManager` subsystem. The `SystemPolicy.framework` is one of the main clients of the `ExecManager` service. You need to search for the `SPExecutionPolicy` class to find true clients of the XPC service. Doing that results in only a single client:

* /System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter

# launchd

The `launchd` plist for `syspolicyd` is located at `/System/Library/LaunchDaemons/com.apple.security.syspolicy.plist`. In addition to all the services we already covered there are a handful of `com.apple.xpc.activity` entries in the plist. You can read more details about what XPC Activities are in the [Energy Efficiency Guide for Mac Apps](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/power_efficiency_guidelines_osx/CentralizedTaskScheduling.html){: target="_blank"}. I am a huge fan of XPC Activities. I think most 3rd party daemons do not make use of this functionality and if they would it would greatly improve their efficiency on the Mac. Taking this approach lets `launchd` make intelligent decision about when to launch certain background activities taking into account things like whether the device is on battery or not or if it can batch up certain requests to be more efficient. The list below just summarizes the ones defined and what they do but does not dive into them in any detail.

## com.apple.security.syspolicy.report

```xml
<key>com.apple.security.syspolicy.report</key>
<dict>
    <key>Delay</key>
    <integer>86400</integer>
    <key>GracePeriod</key>
    <integer>43200</integer>
    <key>Priority</key>
    <string>Maintenance</string>
    <key>AllowBattery</key>
    <false/>
    <key>Repeating</key>
    <true/>
</dict>
```

This job runs once a day on an inexact schedule and never on battery. This gives `launchd` flexibility to batch up this job when the computer is on power and awake. The job makes use of the `ExecutableMeasurementWorker` class and runs the following query in the `/var/db/SystemPolicyConfiguration/ExecPolicy` database:

```sql
SELECT is_signed, file_identifier, bundle_identifier, bundle_version, team_identifier,
signing_identifier, cdhash, main_executable_hash, executable_timestamp, file_size,
is_library,  is_used, responsible_file_identifier, is_valid, is_quarantined
FROM executable_measurements_v2 
WHERE strftime('%W', reported_timestamp, 'unixepoch')/2 < strftime('%W', 'now')/2
ORDER BY reported_timestamp ASC, timestamp ASC
```

It then calls into the private `WirelessDiagnostics.framework` and calls `AWDPostMetric` to log the information.

## com.apple.security.syspolicy.find.bundles

```xml
<key>com.apple.security.syspolicy.find.bundles</key>
<dict>
    <key>Priority</key>
    <string>Maintenance</string>
    <key>AllowBattery</key>
    <false/>
    <key>Interval</key>
    <integer>604800</integer>
    <key>Repeating</key>
    <true/>
</dict>
```

This job runs every 7 days as long as the machine isn't on battery power. It calls into the `BundleFinder` class calling the `searchKnownBundleLocations` method. I didn't look into this in detail but it looks like it attempts to find bundles on the system that it wants to scan and then adds the records into the `scan_targets_v2` table in the `/var/db/SystemPolicyConfiguration/ExecPolicy` database.

## com.apple.security.syspolicy.measure

```xml
<key>com.apple.security.syspolicy.measure</key>
<dict>
    <key>Priority</key>
    <string>Maintenance</string>
    <key>AllowBattery</key>
    <false/>
    <key>Interval</key>
    <integer>259200</integer>
    <key>Repeating</key>
    <true/>
    <key>CPUIntensive</key>
    <true/>
    <key>PowerNap</key>
    <true/>
</dict>
```

This job runs every 3 days and isn't allowed to run on battery. There's not good documentation on all of `launchd`'s keys but I would guess that the `CPUIntensive` flag is an indicator to `launchd` that this job might use a lot of CPU. The `PowerNap` key might indicate that the job can run while the machine is asleep during other PowerNap work periods. This job again makes use of the `ExecutableMeasurementWorker` class this time calling into the `performMeasurementsWithCancelCheck` method. This will in turn run the following sql query:

```sql
SELECT path, responsible_path, is_used, is_library
FROM scan_targets_v2
WHERE strftime('%W', measured_timestamp, 'unixepoch')/2 < strftime('%W', 'now')/2  AND
deferral_count < 5
ORDER BY measured_timestamp ASC, timestamp ASC
```

Based on the scan targets found it looks like the job will then attempt to call `SecStaticCodeCreateWithPath` to create a `SecStaticCode` object and pull out some information around the measured run time of the binary.

## com.apple.security.syspolicy.kext.mt

```xml
<key>com.apple.security.syspolicy.kext.mt</key>
<dict>
    <key>Delay</key>
    <real>518400</real>
    <key>GracePeriod</key>
    <real>43200</real>
    <key>Priority</key>
    <string>Maintenance</string>
    <key>Repeating</key>
    <true/>
</dict>
```

This job runs every 6 days on an inexact schedule. It calls into the `KextManagerPolicy` class calling the `logPolicyState` method. This job will retrieve the current list of KEXTs from the `/var/db/SystemPolicyConfiguration/KextPolicy` database and then for each one attempt to locate it on the file system and eventually call into the `SPMessageTracer` class calling the `logPolicyItem` method. This in turn calls into the `msgtracer_log_with_keys` function which is provided by the `libDiagnosticMessagesClient.dylib`. Based on the name of this job and the initial `KextManagerPolicy` method called it seems to simply be logging the current state of approved KEXTs.

## com.apple.security.syspolicy.rearm

```xml
<key>com.apple.security.syspolicy.rearm</key>
<dict>
    <key>Delay</key>
    <integer>86400</integer>
    <key>GracePeriod</key>
    <integer>3600</integer>
    <key>Priority</key>
    <string>Maintenance</string>
    <key>Repeating</key>
    <true/>
</dict>
```

This job runs once a day. Depending on whether Gatekeeper is enabled and some other internal checks in the `Security.framework` this job will re-enable Gatekeeper if it has been disabled.

# Miscellaneous

## Entitlements

When reviewing a system daemon I'm always interested to see what entitlements it has itself. In the case of `syspolicyd` it has the following entitlements.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.private.iokit.nvram-panicmedic</key>
	<true/>
	<key>com.apple.private.managedclient.configurationprofiles</key>
	<true/>
	<key>com.apple.private.security.storage.SystemPolicyConfiguration</key>
	<true/>
	<key>com.apple.private.storagekitd.info</key>
	<true/>
	<key>com.apple.private.syspolicy.perform-evaluations</key>
	<true/>
	<key>com.apple.rootless.storage.SystemPolicyConfiguration</key>
	<true/>
</dict>
</plist>
```

Nothing really surprising here based on everything described above. It has entitlements that let it access and update the SIP restricted configuration databases for the various services as well as one related to MDM. I'm not sure what the `com.apple.private.iokit.nvram-panicmedic` but I think `syspolicyd` is the only binary on the system with this entitlement.

## Defaults

A lot of system daemons make use of the macOS defaults system to store certain configuration options for itself. Below are the various defaults values I found while reviewing syspolicyd:

### com.apple.security.syspolicy.KextPolicyForceEnable

This system default gets used in the `KextManagerPolicyClass` in the `canLoadKernelExtensionAtURL:isCacheLoad:` method. It checks if the value is set and seems to be a way to potentially force the `KextManager` subsystem on even if other things like SIP is disabled.

### com.apple.security.syspolicy.ExecutableMeasurementDisable

Controls whether or not the `com.apple.security.syspolicy.report` XPC activity job runs.

### com.apple.security.syspolicy.ExecutableMeasurementDisableFullValidation

Controls if full validation is done when performing a measurement.

### com.apple.security.syspolicy.ExecutableMeasurementDisableCalculateFMJ

Whether code signing hash should be calculated during a measurement.

### com.apple.security.GKAutoRearm

If the key is present and has a value of YES, Gatekeeper will turn itself on automatically if it's disabled. If the key is present and has a value of NO, Gatekeeper settings will not change.

## Revision History

What follows is a release history of `syspolicyd` pieced together from the previously open sourced security_systemkeychain code as well as looking through the closed source binary on different versions of macOS. Aside from macOS 10.7 I did not go through each minor OS revision only the most recent version of each major OS release. Where possible I've included links to the source code for each version.

### macOS 10.7.3
* First release of Gatekeeper and `syspolicyd`.
* Only supported [assess](#assess) functionality.

[security_systemkeychain-55105](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55105/syspolicyd/){: target="_blank"}

### macOS 10.7.5
* Added the [update](#update) functionality.

[security_systemkeychain-55119.2](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55119.2/syspolicyd/){: target="_blank"}

### macOS 10.8.4
* No major changes.

[security_systemkeychain-55120.7](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55120.7/syspolicyd/){: target="_blank"}

### macOS 10.9.5
* Added the [record](#record) functionality.

[security_systemkeychain-55191.2](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55191.2/syspolicyd){: target="_blank"}

### macOS 10.10.5
* Last version of `syspolicyd` that was open sourced.
* Added [cancel](#cancel) functionality for long running assessments.

[security_systemkeychain-55202](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55202/syspolicyd/){: target="_blank"}

### macOS 10.11.6
* No major changes.

syspolicyd-9

### macOS 10.12.6
* Added [check-dev-id](#check-dev-id) functionality.

syspolicyd-12.50.1

### macOS 10.13.6
* KextManager subsystem introduced along with user approved KEXT loading.
* ExecManager subsystem introduced tracking 32-bit application launches.
* AppleSystemPolicy.kext introduced along with ExecManager.
* SystemPolicy.framework introduced so clients can query KextManager and ExecManager subsystems.

syspolicyd-45.60.3

### macOS 10.14.x
* ExecManager was expanded to more thoroughly track 32-but application launches.
* Gatekeeper notary service is introduced.
* Added [check-notarized](#check-notarized) functionality.
* Added [ticket-register](#ticket-register) functionality.
* Added [ticket-lookup](#ticket-lookup) functionality.

syspolicyd-45.230.2
