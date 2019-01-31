---
title: syspolicyd internals
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

With my [previous post]({% post_url 2019-01-23-system-policy %}){: target="_blank"} I took a look at the `SystemPolicy.framework` and how it kept track of 32-bit applications that had been run. In the process of looking into that I ended up looking into the internals of `syspolicyd`. Way back in macOS 10.10.5 `syspolicyd` was part of the [security_systemkeychain](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55202/syspolicyd/syspolicyd.cpp.auto.html){: target="_blank"} source code that Apple releases with each version of macOS. Unfortunately since that time `syspolicyd` was moved out of the security_systemkeychain package and closed sourced. This post details the internals of `syspolicyd` as it is today in macOS 10.14.x and covers both what services it provides and what clients connect and use it's functionality.

# Overview

`syspolicyd` was originally introduced in macOS 10.7.3 with the Gatekeeper feature. It's original purpose was to act as the centralized daemon for answering Gatekeeper questions. Today it still serves that purpose but it's scope has greatly expanded. In additioning to assessing applications before running, the daemon also handles authorizing the loading of KEXTs as well as tracking legacy applications that the user has run. In Mojave `syspolicyd` has expanded again and is responsible for handling software notarization checks as well. We'll start with a very high level look at the daemon startup process and then dive deeper into each of `syspolicyd`'s subsystems.  

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

Based on the previously open sourced [syspolicyd.cpp](https://opensource.apple.com/source/security_systemkeychain/security_systemkeychain-55202/syspolicyd/syspolicyd.cpp.auto.html){: target="_blank"} code, we can easily get a sense of how the XPC service works. Essentially the service checks the incoming XPC message dictionary for what `function` is sent over. If the `function` sent over is known then the corresponding function implementation is called. Otherwise if the `function` isn't passed or is unknown then an error is returned to the XPC client. The following `function` values are currently defined as of macOS Mojave:

* [assess](#assess)
* [update](#update)
* [record](#record)
* [cancel](#cancel)
* [check-dev-id](#check-dev-id)
* [check-notarized](#check-notarized)
* [ticket-register](#ticket-register)
* [ticket-lookup](#ticket-lookup)

Most of these functions call into the `Security.framework` to do the real work. Which can be somewhat confusing since we'll see later that the `Security.framework` is one of the primary clients of this service. The framework is set up in such a way that when regular processes call into it's APIs it will send an XPC request to this service and when this service runs it will run the PolicyEngine directly inside of `syspolicyd` using the frameworks code.

#### assess

Calls into `SecAssessmentCreate` passing in the `kSecAssessmentFlagDirect` flag so that the policy engine runs in process. Then calls `SecAssessmentCopyResult` to get the result.

SYSPOLICY_ASSESS_LOCAL();
gEngine().evaluate(path, type, flags, context, result);

/var/db/SystemPolicy is the database for the PolicyEngine in the Security.framework.

#### update

`SecAssessmentCopyUpdate`

result = gEngine().update(target, flags, ctx);

/var/db/SystemPolicy is the database for the PolicyEngine in the Security.framework.

#### record

rax = xpc_connection_copy_entitlement_value(arg2, "com.apple.private.assessment.recording", arg2);

if (!SecAssessmentControl(CFSTR("ui-record-reject-local"), (void *)info.get(), &errors)) {

gEngine().recordFailure(CFDictionaryRef(arguments));

/var/db/SystemPolicy is the database for the PolicyEngine in the Security.framework.

#### cancel

Cancel an in progress assessment request

#### check-dev-id

rax = SecAssessmentControl(@"ui-get-devid-local", &var_18, &var_10);

/var/db/SystemPolicy

CFBooleanRef &result = *(CFBooleanRef*)(arguments);
if (gEngine().value<int>("SELECT disabled FROM authority WHERE label = 'Developer ID';", true))
    result = kCFBooleanFalse;
else
    result = kCFBooleanTrue;
return true;

#### check-notarized

rax = SecAssessmentControl(@"ui-get-notarized-local", &var_18, &var_10);

CFBooleanRef &result = *(CFBooleanRef*)(arguments);
if (gEngine().value<int>("SELECT disabled FROM authority WHERE label = 'Notarized Developer ID';", true))
    result = kCFBooleanFalse;
else
    result = kCFBooleanTrue;
return true;

#### ticket-register

 rbx = [[TicketService globalService] retain];
    r14 = [rbx registerTicket:arg0];
    [rbx release];
    rax = r14;
    return rax;

/var/db/SystemPolicyConfiguration/Tickets

@"INSERT INTO tickets (hash, hash_type, timestamp, flags) VALUES (?1, ?2, ?3, ?4)"

#### ticket-lookup

/var/db/SystemPolicyConfiguration/Tickets


rbx = [[TicketService globalService] retain];
r14 = [rbx lookupTicketByHash:arg0 withHashType:arg1 withOptions:arg2 ticketDate:arg3];
[rbx release];
rax = r14;
return rax;

CKTicketStore
            objc_storeStrong(*ivar_offset(_container) + rbx, @"com.apple.gk.ticket-delivery");
            objc_storeStrong(&rbx->_environment, @"production");
https://api.apple-cloudkit.com/database/1/%@/%@/public/records/lookup
https://api.apple-cloudkit.com/database/1/com.apple.gk.ticket-delivery/production/public/records/lookup

or pull from local database
SELECT tickets.flags  FROM hashes INNER JOIN tickets ON hashes.ticket_id = tickets.id  WHERE hashes.hash = ?1 AND hashes.hash_type = ?2

## Clients

### Security.framework

The `Security.framework` is the primary client of this service with all of it's SecAssessment APIs. As mentioned above if you call into these APIs outside of `syspolicyd` then the framework will use XPC to request the information from `syspolicyd`. Here's a short overview of the SecAssessment APIs:

| API                         | Description                                                |
|-----------------------------|------------------------------------------------------------|
| SecAssessmentCreate         | Ask the system for its assessment of a proposed operation. |
| SecAssessmentCopyUpdate     | Make changes to the system policy configuration.           |
| SecAssessmentControl        | Miscellaneous system policy operations.                    |
| SecAssessmentTicketRegister | Registering stapled ticket with system.                    |
| SecAssessmentTicketLookup   | Check whether software is notarized.                       |

Luckily `Security.framework` is still open sourced, so you can look more into the internals of these APIs here:

[https://opensource.apple.com/tarballs/Security/Security-58286.220.15.tar.gz](https://opensource.apple.com/tarballs/Security/Security-58286.220.15.tar.gz){: target="_blank"}

This means that if we really want to find out what clients use this service we should search the system for users of the SecAssessment APIs. I used ripgrep on a Mojave system to search for binaries that used `SecAssessmentCreate` and found the following items:
	
* /usr/sbin/spctl
* /System/Library/CoreServices/Installer.app/Contents/MacOS/Installer
* /System/Library/Frameworks/Automator.framework/Versions/A/Automator
* /System/Library/PrivateFrameworks/PackageKit.framework/Versions/A/PackageKit
* /System/Library/PrivateFrameworks/XprotectFramework.framework/Versions/A/XPCServices/XprotectService.xpc/Contents/MacOS/XprotectService

# KextManager

The KextManager subsystem is responsible for [user-approved kernel extension loading](https://developer.apple.com/library/archive/technotes/tn2459/_index.html){: target="_blank"}. The following classes make up the core of the KextManager subsystem:

* KextManagerService
* KextManagerPolicy
* KextManagerDatabase
* KextManagerUI

![KextManager Subsystem](/images/syspolicyd-internals-1.png){: .align-center}

Some more descriptions about how all this works.

## Services

### com.apple.security.syspolicy.kext

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

Any user
- (void)copyPendingApprovalsWithReply:(void (^)(NSArray *, NSError *))arg1;
- (void)copyCurrentPolicyWithReply:(void (^)(NSArray *, NSError *))arg1;
- (void)teamIdentifierIsAllowed:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;

kextd
- (void)canLoadKernelExtensionInCache:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
com.apple.rootless.kext-secure-management

- (void)canLoadKernelExtension:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
com.apple.rootless.kext-secure-management

remoteservice
- (void)updatePolicyItems:(NSArray *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
com.apple.private.security.syspolicy.kext-policy-management

ExecutionPolicyService
- (void)setUserApprovalAllowed:(BOOL)arg1 withReply:(void (^)(BOOL, NSError *))arg2;
com.apple.private.security.syspolicy.mdm-kext-policy-management

- (void)installMDMPayload:(NSString *)arg1 withTeams:(NSArray *)arg2 andExtensions:(NSDictionary *)arg3 withReply:(void (^)(BOOL, NSError *))arg4;
com.apple.private.security.syspolicy.mdm-kext-policy-management"

- (void)removeMDMPayload:(NSString *)arg1 withReply:(void (^)(BOOL, NSError *))arg2;

com.apple.private.security.syspolicy.mdm-kext-policy-management

## Clients

### /sbin/kextload
### /usr/sbin/kextcache
### /usr/libexec/kextd
### /usr/bin/kextutil
### SystemPolicy.framework
/System/Library/PrivateFrameworks/SystemPolicy.framework/Versions/A/SystemPolicy
    /sbin/kextload
    /usr/libexec/syspolicyd
    /usr/libexec/amfid
    /usr/sbin/kextcache
    /usr/libexec/kextd
    /usr/sbin/spctl
    /usr/bin/kextutil
    /System/Library/SystemProfiler/SPDisabledApplicationsReporter.spreporter/Contents/MacOS/SPDisabledApplicationsReporter
    /System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter
    /System/Library/PreferencePanes/Security.prefPane/Contents/MacOS/Security
    /System/Library/PrivateFrameworks/OSInstaller.framework/Versions/A/OSInstaller
    /System/Library/PrivateFrameworks/ConfigurationProfiles.framework/XPCServices/ExecutionPolicyService.xpc/Contents/MacOS/ExecutionPolicyService

# ExecManager

## Services

```c
@interface ExecManagerService : NSObject <NSXPCListenerDelegate, ExecManager>
{
    unsigned int _driver;
    NSObject<OS_dispatch_queue> *_mig_queue;
    NSObject<OS_dispatch_source> *_mig_source;
    NSObject<OS_dispatch_queue> *_work_queue;
    NSObject<OS_dispatch_semaphore> *_work_semaphore;
    NSXPCListener *_listener;
    NSXPCInterface *_interface;
    NSObject<OS_dispatch_queue> *_completionQueue;
    ExecManagerPolicy *_policy;
    ExecManagerDatabase *_db;
    NSObject<OS_dispatch_queue> *_measurement_queue;
    ExecutableMeasurementWorker *_measurementWorker;
    BundleFinder *_bundleFinder;
    BOOL _measurementsEnabled;
    AWDServerConnection *_awdServer;
}

+ (id)globalManager;
- (void).cxx_destruct;
- (void)logExecutable:(id)arg1 withResponsiblePath:(id)arg2 isExecution:(BOOL)arg3;
- (void)registerActivities;
- (void)logOldPlatformBinary:(id)arg1 withSigningID:(id)arg2 withCdhash:(id)arg3 withCodesignFlags:(unsigned long long)arg4 isExecution:(BOOL)arg5;
- (void)copyLegacyExecutionHistoryWithReply:(CDUnknownBlockType)arg1;
- (BOOL)listener:(id)arg1 shouldAcceptNewConnection:(id)arg2;
- (BOOL)startXPCService;
- (void)notify32bitExecForUser:(unsigned int)arg1 withPID:(int)arg2 withProcessPath:(id)arg3 withProcessTeamID:(id)arg4 withProcessSigningID:(id)arg5 withLibraryPath:(id)arg6 withLibraryTeamID:(id)arg7 withLibrarySigningID:(id)arg8 withCdhash:(id)arg9 forEvaluationID:(long long)arg10;
- (void)sendEvaluationResult:(int)arg1 forEvaluationID:(long long)arg2;
- (BOOL)startMIGService;
- (id)init;

// Remaining properties
@property(readonly, copy) NSString *debugDescription;
@property(readonly, copy) NSString *description;
@property(readonly) unsigned long long hash;
@property(readonly) Class superclass;

@end
```

### com.apple.security.AppleSystemPolicy.mig

You won't find a reference to this string anywhere else on the system though since this is registered as a HostSpecialPort in the launchd config. That means if your root you can get access to this system by doing the following

host_get_host
host_get_special_port(29)

[Hopper script](https://github.com/knightsc/hopper/blob/master/scripts/MIG%20Detect.py){: target="_blank"} to detect MIG subsystems

```
0x10003ad70: MIG Subsystem 18600: 4 messages
0x10003ad98: MIG msg 18600
0x10003adc0: MIG msg 18601
0x10003ade8: MIG msg 18602
0x10003ae10: MIG msg 18603
```
#### notify_32bit_exec

r12 = [[ExecManagerService globalManager] retain];
rsi = @selector(notify32bitExecForUser:withPID:withProcessPath:withProcessTeamID:withProcessSigningID:withLibraryPath:withLibraryTeamID:withLibrarySigningID:withCdhash:forEvaluationID:);


#### notify_32bit_mmap

r13 = [[ExecManagerService globalManager] retain];
rsi = @selector(notify32bitExecForUser:withPID:withProcessPath:withProcessTeamID:withProcessSigningID:withLibraryPath:withLibraryTeamID:withLibrarySigningID:withCdhash:forEvaluationID:);
    
#### log_platform_binary

rbx = [[ExecManagerService globalManager] retain];
[rbx logOldPlatformBinary:r14 withSigningID:r13 withCdhash:r12 withCodesignFlags:var_30 isExecution:var_29];

#### log_executable

rbx = [[ExecManagerService globalManager] retain];
[rbx logExecutable:var_38 withResponsiblePath:r15 isExecution:r14];

### com.apple.security.syspolicy.exec

```c
@protocol ExecManager
- (void)copyLegacyExecutionHistoryWithReply:(void (^)(NSArray *, NSError *))arg1;
@end
```

## Clients

### AppleSystemPolicy.kext

This is the only client of the mig service. Which should be expected based on the naming. 
How to search for host_special_port use though?

$ sysctl -a | grep asp
security.mac.asp.cache_entry_count: 3417
security.mac.asp.cache_allocation_count: 4104
security.mac.asp.cache_release_count: 687
security.mac.asp.exec_hook_time: 616061
security.mac.asp.exec_hook_count: 252162
security.mac.asp.library_hook_time: 2385933
security.mac.asp.library_hook_count: 1051821

### SystemPolicy.framework
/System/Library/PrivateFrameworks/SystemPolicy.framework/Versions/A/SystemPolicy
    /sbin/kextload
    /usr/libexec/syspolicyd
    /usr/libexec/amfid
    /usr/sbin/kextcache
    /usr/libexec/kextd
    /usr/sbin/spctl
    /usr/bin/kextutil
    /System/Library/SystemProfiler/SPDisabledApplicationsReporter.spreporter/Contents/MacOS/SPDisabledApplicationsReporter
    /System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter
    /System/Library/PreferencePanes/Security.prefPane/Contents/MacOS/Security
    /System/Library/PrivateFrameworks/OSInstaller.framework/Versions/A/OSInstaller
    /System/Library/PrivateFrameworks/ConfigurationProfiles.framework/XPCServices/ExecutionPolicyService.xpc/Contents/MacOS/ExecutionPolicyService

# launchd

/System/Library/LaunchDaemons/com.apple.security.syspolicy.plist 

com.apple.xpc.activity

Apple energy effecient guide


```c
/*!
 * @constant XPC_ACTIVITY_CHECK_IN
 * This constant may be passed to xpc_activity_register() as the criteria
 * dictionary in order to check in with the system for previously registered
 * activity using the same identifier (for example, an activity taken from a
 * launchd property list).
 */
const xpc_object_t XPC_ACTIVITY_CHECK_IN;


xpc_activity_register("identifier", XPC_ACTIVITY_CHECK_IN, ^(xpc_activity_t activity) {
    /* do background or deferred work */                                   
});
```

register_xpc_activities
https://developer.apple.com/library/archive/documentation/Performance/Conceptual/power_efficiency_guidelines_osx/CentralizedTaskScheduling.html

All run at priority maintenance

XPC_ACTIVITY_PRIORITY_MAINTENANCE. Maintenance-level priority. Maintenance priority is intended for user-invisible maintenance tasks such as garbage collection or optimization.



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

Once a day inexact
If _measurementsEnabled is enabled

ExecutableMeasurementWorker
[_measurementWorker)) reportMeasurements]

/var/db/SystemPolicyConfiguration/ExecPolicy

SELECT is_signed, file_identifier, bundle_identifier, bundle_version, team_identifier,  signing_identifier, cdhash, main_executable_hash, executable_timestamp, file_size, is_library,  is_used, responsible_file_identifier, is_valid, is_quarantined  FROM executable_measurements_v2  WHERE strftime('%W', reported_timestamp, 'unixepoch')/2 < strftime('%W', 'now')/2  ORDER BY reported_timestamp ASC, timestamp ASC

/System/Library/PrivateFrameworks/WirelessDiagnostics.framework/Versions/A/WirelessDiagnostics

AWDPostMetric

NSUserDefaults
com.apple.security.syspolicy
ExecutableMeasurementDisable

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

Every 7 days
not on battery

/var/db/SystemPolicyConfiguration/ExecPolicy

BundleFinder
_bundleFinder)) searchKnownBundleLocations];

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

Every 3 days
CPUIntensive and PowerNap but not on Battery

ExecutableMeasurementWorker
_measurementWorker performMeasurementsWithCancelCheck

/var/db/SystemPolicyConfiguration/ExecPolicy

SELECT path, responsible_path, is_used, is_library  FROM scan_targets_v2  WHERE strftime('%W', measured_timestamp, 'unixepoch')/2 < strftime('%W', 'now')/2  AND deferral_count < 5  ORDER BY measured_timestamp ASC, timestamp ASC

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

Every 6 days inexact

KextManagerPolicy
(_policy)) logPolicyState]

/var/db/SystemPolicyConfiguration/KextPolicy

SELECT DISTINCT team_id FROM kext_policy WHERE team_id IS NOT NULL

SPMessageTracer */
+(void)logPolicyItem:(void *)arg2 forBundle:(void *)arg3 forDeveloper:(void *)arg4 forCdhash:(void *)arg5 approved:(char)arg6 migrated:(char)arg7 panicMedicDisabled:(char)arg8 {
    r14 = arg3;
    var_2A = arg8;

/usr/lib/libDiagnosticMessagesClient.dylib

 msgtracer_log_with_keys("com.apple.syspolicy.kext.policy_item", 0x5, "com.apple.message.approved", r13, "com.apple.message.bundle_id", var_50, "com.apple.message.cdhash", var_48, "com.apple.message.developer_name", var_38, "com.apple.message.team_id", r12, "com.apple.message.was_migrated", rbx, "com.apple.message.was_panic_medic_disabled", r15);
    

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

Once a day inexact

$ sudo defaults read /Library/Preferences/com.apple.security 
{
    SecItemSynchronizable = 1;
}

```c
int syspolicy_rearm_internal() {
    r15 = CFPreferencesCopyValue(@"GKAutoRearm", @"com.apple.security", **_kCFPreferencesAnyUser, **_kCFPreferencesCurrentHost);
    rax = *_kCFBooleanFalse;
    rbx = *rax;
    if (rbx != r15) {
            rax = CFPreferencesAppValueIsForced(@"EnableAssessment", @"com.apple.systempolicy.control");
            if (rax != 0x1) {
                    rax = CFPreferencesAppValueIsForced(@"AllowIdentifiedDevelopers", @"com.apple.systempolicy.control");
                    if (rax != 0x1) {
                            rax = SecAssessmentControl(@"ui-status", &var_28, 0x0);
                            if ((rax != 0x0) && (var_28 == rbx)) {
                                    rax = SecAssessmentControl(@"rearm-status", &var_20, 0x0);
                                    if (rax != 0x0) {
                                            xmm0 = intrinsic_movsd(xmm0, var_20);
                                            xmm0 = intrinsic_ucomisd(xmm0, *double_value_2_592E06);
                                            if (xmm0 > 0x0) {
                                                    SecAssessmentControl(@"ui-enable", 0x0, 0x0);
                                                    SecAssessmentControl(@"ui-enable-devid", 0x0, 0x0);
                                                    rax = notify_post("com.apple.security.assessment.rearm");
                                            }
                                    }
                            }
                    }
            }
    }
    if (r15 != 0x0) {
            rax = CFRelease(r15);
    }
    return rax;
}
```

# Miscellaneous

## Entitlements

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

## Defaults

### com.apple.security.syspolicy.KextPolicyForceEnable

int sub_100010845() {
    rbx = [[NSUserDefaults alloc] initWithSuiteName:@"com.apple.security.syspolicy"];
    r14 = [rbx boolForKey:@"KextPolicyForceEnable"];
    [rbx release];
    rax = sign_extend_64(r14);
    return rax;
}

blank on a clean machine

Used in  -[KextManagerPolicy canLoadKernelExtensionAtURL:isCacheLoad:]:

### com.apple.security.syspolicy.ExecutableMeasurementDisable
NSUserDefaults
com.apple.security.syspolicy
ExecutableMeasurementDisable

Controls whether or not the `com.apple.security.syspolicy.report` XPC activity job runs or not

### com.apple.security.syspolicy.ExecutableMeasurementDisableFullValidation

Controls if full validation is done when performing measurement

### com.apple.security.syspolicy.ExecutableMeasurementDisableCalculateFMJ

Whether code signing hash should be calculated during measurement

### com.apple.security.GKAutoRearm

Seems to be defaults to off. If it's enabled

defaults write /Library/Preferences/com.apple.security GKAutoRearm -bool true

Then Gate Keeper will be ren-enabled every time the XPC job runs

If the key is present and has a value of YES, Gatekeeper will turn itself on automatically if it's disabled. If the key is present and has a value of NO, Gatekeeper settings will not change.

## Revision History

What follows is a release history of `syspolicyd` pieced together from the oreviously open sourced security_systemkeychain code as well as looking through the closed source binary on different versions of macOS. Aside frm macOS 10.7 I did not go through each minor OS revision only the most recent version of each major OS release. Where possible I've included links to the source code for each version.

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
* Added [cancel](#cancel) functionality to long running assessments.

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
* ExecManager was expanded to more thouroughly track 32-but application launches.
* Gatekeeper notary service is introduced.
* Added [check-notarized](#check-notarized) functionality.
* Added [ticket-register](#ticket-register) functionality.
* Added [ticket-lookup](#ticket-lookup) functionality.

syspolicyd-45.230.2
