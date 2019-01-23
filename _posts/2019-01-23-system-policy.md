---
title: macOS and 32-bit applications
categories:
  - Reverse Engineering
tags:
  - Assembly
  - Go
  - macOS
  - MacAdmins
  - osquery
  - x86
---

At the 2018 WWDC State of the Union event, Apple vice president of software Sebastien Marineau revealed Mojave will be "the last release to support 32-bit at all". Since macOS 10.13.4, Apple has provided the ability to set your machine to 64-bit only mode for testing. For most users this is not a very convenient way to test. As of 10.14 the System Information application has a new "Legacy Software" section that shows you all of the 32-bit applications that have been run on the machine. This new "Legacy Software" information provides great insight for Mac Admins into what 32-bit applications their users are running so that they can work with vendors to get software updated prior to the release of macOS 10.15. From an admin perspective, it would be nice to be able to get this information in an automated way. This post covers how I went about digging into this new feature and exposing it in a way that could be queried from [osquery](https://osquery.io/){: target="_blank"}.

# System Information

I started my investigation with the `/Applications/Utilities/System Information.app/` application. You can see a screenshot of the new "Legacy Software" section below:

![System Information](/images/system-policy-1.png){: .align-center}

Initially I opened up this application in [Hopper](https://www.hopperapp.com/){: target="_blank"} and started searching through the strings for things with the word "Legacy" in it. This didn't turn up anything useful. Next I decided to look through the different files that the application had open. You can use the built in `Activity Monitor` utility to pull up a list of processes and then you can double click on the process to open additional information. Looking through the `Open Files and Ports` tabs for the `System Information` application turned up a file that mentioned "Legacy Software"

```
/System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter
```

Looking into the `/System/Library/SystemProfiler` folder shows a handful of `.spreporter` files. These are bundle plugins that get used by the `System Information` application to provide the information for the different sections in the application. So next I decided to look at the `SPLegacySoftwareReporter.spreporter` bundle.

# SPLegacySoftwareReporter

Again I opened the `SPLegacySoftwareReporter` in [Hopper](https://www.hopperapp.com/){: target="_blank"} and started searching through the binary for strings with the word "Legacy" in it. I immediately came across the following functiion:

```c
int _SPLegacySoftwareReporter_updateDictionary() {
    rax = [rdi retain];
    var_518 = rax;
    r14 = [[rax objectForKey:@"_items"] retain];
    rbx = [[SPExecutionPolicy alloc] init];
    r12 = [[rbx legacyExecutionHistory] retain];
    ...
    ...
```

Searching for the `legacyExecutionHistory` method in the `SPLegacySoftwareReporter` turned up nothing which means it's probably provided by a different framework. Doing a Google search for `SPExecutionPolicy` to see if Apple had any documentation on that class also turned up nothing. So next I looked at what libraries the `SPLegacySoftwareReporter` bundle depended on using [jtool2](http://newosxbook.com/tools/jtool2.tgz){: target="_blank"}. You could also use the built-in `otool` command for this but [jtool2](http://newosxbook.com/tools/jtool2.tgz){: target="_blank"} tends to be my go to swiss army knife for inspecting binaries from the command line.

```shell
$ jtool2 -L /System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter 
/System/Library/SystemProfiler/SPLegacySoftwareReporter.spreporter/Contents/MacOS/SPLegacySoftwareReporter:
	/System/Library/PrivateFrameworks/SPSupport.framework/Versions/A/SPSupport (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/PrivateFrameworks/SystemPolicy.framework/Versions/A/SystemPolicy (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 1555.10.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
	/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 1555.10.0)
	/System/Library/Frameworks/CoreServices.framework/Versions/A/CoreServices (compatibility version 1.0.0, current version 933.0.0)
```

Looking through this output I noticed that there's a `SystemPolicy.framework` which would seem to match up with the class name prefix used in the SPExecutionPolicyClass` I found above.

# SystemPolicy.framework

With the `SystemPolicy.framework` open in [Hopper](https://www.hopperapp.com/){: target="_blank"} I started looking through the `SPExecutionPolicy` class.

```c
/* @class SPExecutionPolicy */
-(void *)init {
    var_48 = self;
    *(&var_48 + 0x8) = *0xea30;
    rax = [[&var_48 super] init];
    rbx = rax;
    if (rbx != 0x0) {
        var_30 = [SPExecutionHistoryItem class];
        [NSString class];
        [NSDate class];
        [NSArray class];
        r14 = [[NSSet setWithObjects:var_30] retain];
        rax = [NSXPCInterface interfaceWithProtocol:@protocol(ExecManager)];
        rax = [rax retain];
        r13 = *ivar_offset(_interface);
        rdi = *(rbx + r13);
        *(rbx + r13) = rax;
        [rdi release];
        [*(rbx + r13) setClasses:r14 forSelector:@selector(copyLegacyExecutionHistoryWithReply:) argumentIndex:0x0 ofReply:0x1];
        rax = [NSXPCConnection alloc];
        rax = [rax initWithMachServiceName:@"com.apple.security.syspolicy.exec" options:0x1000];
        r12 = *ivar_offset(_connection);
        rdi = *(rbx + r12);
        *(rbx + r12) = rax;
        [rdi release];
        [*(rbx + r12) setRemoteObjectInterface:*(rbx + r13)];
        [*(rbx + r12) resume];
        [r14 release];
    }
    rax = rbx;
    return rax;
}
```

Hopper does a great job with it's pseudo-code mode allowing you to glance at a high level approximation of the disassembled code. I did a little manual clean up on it and ended up with reversed code like this:

```c
- (instancetype)init {
    self = [super init];
    if (self) {
        self.interface = [NSXPCInterface interfaceWithProtocol:ExecManager];
        self.connection = [[NSXPCConnection alloc] initWithMachServiceName:@"com.apple.security.syspolicy.exec"
                                                                   options:0x1000];
        [self.connection setRemoteObjectInterface:self.interface];
        [self.connection resume];
    }
    return self
}
```

So the `SPExecutionPolicy` class is just using XPC to ask something else for the data. On macOS, `launchd` is responsible for defining these mach services, so I searched through files in `/System` to find what `launchd` daemon is doing the real work.

```shell
$ cd /System
$ rg "com.apple.security.syspolicy.exec" * 2> /dev/null
Library/LaunchDaemons/com.apple.security.syspolicy.plist
20:            <key>com.apple.security.syspolicy.exec</key>
```

Looking at this `launchd` plist you see the following `ProgramArguements` key:

```
<key>ProgramArguments</key>
<array>
    <string>/usr/libexec/syspolicyd</string>
</array>
```

# syspolicyd

One note before I continue down this rabbit hole. At this point I could have stopped reverse engineering. It should be enough to simply know that I can use the `SystemPolicy.framework` and the `SPExecutionPolicy` class to get a list of items back, but I wanted to know more about how and where this information was actually stored on the system.

I started by looking at the `main` function of the daemon. After pulling the pseudo code out of [Hopper](https://www.hopperapp.com/){: target="_blank"} and doing some clean up it looks like this:

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

So the `ExecManagerService` seems like a good starting point but I also just wanted to see what other related classes there might be. For this I turned to jtool again.

```shell
$ jtool -d objc /usr/libexec/syspolicyd | grep Exec
ExecManagerPolicy
ExecutableMeasurementWorker
ExecManagerService
ExecutableMeasurement
ExecutableMeasurementBuilder
AWDSystemPolicyExecutableMeasurement
ExecManagerDatabase
SPExecutionHistoryItem
```

The `ExecManagerDatabase` class also looked interesting. After looking at that class and seeing where it was used I saw that it was created from the `ExecManagerService` class used in the daemon main method. Here's a snippet of relevant code:

```c
rbx = [[NSURL fileURLWithPath:@"/var/db/SystemPolicyConfiguration/ExecPolicy"] retain];
rax = *ivar_offset(_listener);
rdi = *(r12 + rax);
*(r12 + rax) = 0x0;
[rdi release];
rax = *ivar_offset(_interface);
rdi = *(r12 + rax);
*(r12 + rax) = 0x0;
[rdi release];
rax = dispatch_get_global_queue(0x19, 0x0);
rax = [rax retain];
rcx = *ivar_offset(_completionQueue);
rdi = *(r12 + rcx);
*(r12 + rcx) = rax;
[rdi release];
var_38 = rbx;
rdx = rbx;
rbx = r14;
rax = [ExecManagerDatabase databaseWithURL:rdx];
rax = [rax retain];
rcx = *ivar_offset(_db);
rdi = *(r12 + rcx);
*(r12 + rcx) = rax;
```

When the `ExecManagerService` class is initialized it also creates an instance of the `ExecManagerDatabase` class passing in a `NSURL` object with the path `/var/db/SystemPolicyConfiguration/ExecPolicy`. Taking a quick look at this file it appears to be a SQLite database.

```shell
$ sudo file /var/db/SystemPolicyConfiguration/ExecPolicy
/var/db/SystemPolicyConfiguration/ExecPolicy: SQLite 3.x database, last written using SQLite version 3024000
```

I used `sqlite3` to open up and inspect this database:

```shell
$ sudo sqlite3 /var/db/SystemPolicyConfiguration/ExecPolicy ".schema"
CREATE TABLE settings ( name TEXT, value TEXT, PRIMARY KEY (name) );
CREATE TABLE old_platform_cache ( key TEXT, ts INTEGER, PRIMARY KEY (key) );
CREATE TABLE legacy_exec_history_v4 (  uid INTEGER NOT NULL,  exec_path TEXT NOT NULL,  mmap_path TEXT,  signing_id TEXT,  team_id TEXT,  cd_hash TEXT,  responsible_path TEXT,  developer_name TEXT,  last_seen TEXT NOT NULL DEFAULT (datetime('now')),  PRIMARY KEY (uid, exec_path, mmap_path));
CREATE TABLE scan_targets_v2 (  path TEXT NOT NULL,  responsible_path TEXT,  is_library INTEGER,  is_used INTEGER,  timestamp INTEGER NOT NULL DEFAULT (strftime('%s','now')),  measured_timestamp INTEGER,  deferral_count INTEGER,  PRIMARY KEY (path));
CREATE TABLE executable_measurements_v2 (  is_signed INTEGER,  file_identifier TEXT NOT NULL,  bundle_identifier TEXT,  bundle_version TEXT,  team_identifier TEXT,  signing_identifier TEXT,  cdhash TEXT NOT NULL,  main_executable_hash TEXT,  executable_timestamp INTEGER,  file_size INTEGER,  is_library INTEGER,  is_used INTEGER,  responsible_file_identifier TEXT,  is_valid INTEGER,  is_quarantined INTEGER,  timestamp INTEGER NOT NULL DEFAULT (strftime('%s','now')),  reported_timestamp INTEGER,  PRIMARY KEY (cdhash));
```

The `/var/db` location is protected by SIP, so a regular user cann't read from the database. With this information I now knew what application was responsible for collecting the 32-bit "Legacy Software" information and where the information was stored. Based on all of this I set out to create a way to get this information using [osquery](https://osquery.io/){: target="_blank"}.

# osquery

If you're not familiar with osquery then I highly recommend you take a look at their webpage.

[https://osquery.io/](https://osquery.io/){: target="_blank"}

Essentially it's a utility application that lets you use SQL commands to query information about your computer. Additionally you can deploy it across your entire fleet of computers and then use it to feed information back towards a central repository of device information. What's even better is it provides an easy way to create extensions and add support for new information. The official osquery documentation has a good overview of the process.

[osquery SDK and Extensions](https://osquery.readthedocs.io/en/stable/development/osquery-sdk/){: target="_blank"}

In this case I chose to use the [Go](https://github.com/kolide/osquery-go){: target="_blank"} bindings provided by [Kolide](https://kolide.com){: target="_blank"} to write my extension.

On my first pass at writing an extension I created one that used a SQLite library to query the information from the `ExecPolicy` policy database directly. This worked just fine but did have the drawback that since the SQLite database is in a SIP protected directory it only worked if I ran osquery as root. Then I realized that if I used the `SystemPolicy.framework` directly it would communicate over XPC and allow me to get the information in the SQLite database as a regular user. You can download my extension from my github page here:

[https://github.com/knightsc/system_policy](https://github.com/knightsc/system_policy){: target="_blank"}

The extension also supports querying kext authorization information since that's also part of the `SystemPolicy.framework` Here's some examples of the types of queries you can run with the extension in osquery.

```sql
-- Get a list of 32-bit applications that have been run
select * from legacy_exec_history;

-- Get a list of kexts that need to be approved
select * from kext_policy where allowed = 0;

-- Get a list of kexts that have already been approved
select * from kext_policy where allowed = 1;
```

Hopefully this post gave you a better idea of how you can dig into Apple's provided utilitys to find better ways to automate the gathering of that information across your entire fleet.
