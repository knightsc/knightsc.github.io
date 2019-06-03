---
title: Debugging Apple binaries that use PT_DENY_ATTACH
categories:
  - Debugging
tags:
  - Kernel
  - LLDB
  - macOS
  - Python
---

Recently while looking into the Apple `adid` daemon, I noticed that I couldn't attach to the process with `lldb` even if SIP was completely disabled. After digging into it a little bit I came to the conclusion that `adid` was calling the `ptrace` API passing in `PT_DENY_ATTACH`. There are numerous other posts out there (like [this one](https://alexomara.com/blog/defeating-anti-debug-techniques-macos-ptrace-variants/){: target="_blank"}) that talk about defeating `PT_DENY_ATTACH` if you're running the application yourself. In my case `adid` is started as a LaunchDaemon and is already running by the time a user is logged in. I decided to take a look at how you could defeat the `ptrace` call even after the application is already running.

First a quick note on how I determined that `adid` was using the `ptrace` API. I started off on a virtual machine with SIP disabled. This is my usual setup when inspecting Apple platform binaries. I tried to connect to `adid` using `lldb` but it didn't seem to work.

```shell
$ ps xa | grep adid
  374   ??  Ss     0:00.06 /System/Library/PrivateFrameworks/CoreADI.framework/adid

$ lldb -o "attach 374"
(lldb) attach 374
error: attach failed: Error 1

$ sudo lldb -o "attach 374"
Password:
(lldb) attach 374
error: attach failed: lost connection
```
The second attempt running `lldb` with `sudo` resulted in `debugserver` itself crashing with a segmentation fault. This is what made me think that `ptrace` might be in use. Checking the symbols on `adid` confirmed this.

```shell
$ jtool2 -S /System/Library/PrivateFrameworks/CoreADI.framework/adid  | grep ptrace
                 U _ptrace
```

From there I decided that I should be able to either catch the `ptrace` call at boot time by debugging XNU itself or potentially inspect and undo the `ptrace` call from the running process. I set up my virtual machine for [kernel debugging]({% post_url 2018-08-15-macos-kernel-debugging %}){: target="_blank"} and then decided to look into what the `ptrace` call itself does when called. Looking in the [XNU source](https://github.com/knightsc/darwin-xnu/blob/master/bsd/kern/mach_process.c#L149){: target="_blank"} we can see that the following happens when calling `ptrace(PT_DENY_ATTACH, 0, 0, 0);`

```c
...
#define P_LNOATTACH     0x00001000
...
if (uap->req == PT_DENY_ATTACH) {
  proc_lock(p);
  if (ISSET(p->p_lflag, P_LTRACED)) {
    proc_unlock(p);
    KERNEL_DEBUG_CONSTANT(BSDDBG_CODE(DBG_BSD_PROC, BSD_PROC_FRCEXIT) | DBG_FUNC_NONE,
              p->p_pid, W_EXITCODE(ENOTSUP, 0), 4, 0, 0);
    exit1(p, W_EXITCODE(ENOTSUP, 0), retval);

    thread_exception_return();
    /* NOTREACHED */
  }
  SET(p->p_lflag, P_LNOATTACH);
  proc_unlock(p);

  return(0);
}
...
```

When the `ptrace` API is called with the `PT_DENY_ATTACH` argument what happens is that the `proc` structures `p_lflag` field gets updated to have the `P_LNOATTACH` flag set. Some of the `proc` fields are [exposed to user space](https://github.com/knightsc/darwin-xnu/blob/master/bsd/kern/proc_info.c#L615){: target="_blank"} APIs but unfortunately it doesn't look like the `p_lflag` is. At this point I decided to start up `lldb` and connect to the kernel and inspect the task itself. (The following commands assume you've loaded the lldbmacros that come with the Kernel Debug Kit)

First I got some information about the `adid` task.

```shell
(lldb) showtask -F adid
task                 vm_map               ipc_space            #acts flags    pid       process             io_policy  wq_state  command
0xffffff8017de1188   0xffffff8018d97838   0xffffff801acca980       3 R        374   0xffffff801acecd50            BLT  2  2  0    adid

(lldb) showprocinfo 0xffffff801acecd50
Process 0xffffff801acecd50
	name adid
	pid:374    task:0xffffff8017de1188   p_stat:2      parent pid: 1
Cred: euid 265 ruid 265 svuid 265
Flags: 0x4004
	0x00000004 - process is 64 bit
	0x00004000 - process has called exec
```

Now that we have an address for the process itself we can inspect the `p_lflag` field.

```shell
(lldb) p/x ((struct proc *)0xffffff801acecd50)->p_lflag
(unsigned int) $2 = 0x00801000
```

You can see that the `p_lflag` field has the `P_LNOATTACH` flag set. All we need to do to effectively negate the `ptrace` call is to change the value of the `p_lflag` field.

```shell
(lldb) expr ((struct proc *)0xffffff801acecd50)->p_lflag = 0x00800000
```

After this `lldb` can happily attach to the `adid` process. This effectively solves the original problem I had, which was being able to debug `adid` after it has already started, but it got me wondering what other processes Apple has that might be doing the same type of thing. I created a short [python script](https://github.com/knightsc/lldb/blob/master/scripts/kernhelper.py){: target="_blank"} to check all tasks to see if they had the `P_LNOATTACH` flag set.

```shell
(lldb) command script import kernhelper.py

(lldb) script kernhelper.show_ptrace_deny_attach_procs()
...
...
0xffffff8025cabe20: SecurityAgent
0xffffff802a615b70: authorizationhos
0xffffff802cbd87f0: adid

(lldb)
```

So you can see there are a couple other processes running after first login that also have called `ptrace` passing in the `PT_DENY_ATTACH` argument. These correspond to the following binaries:

```shell
/System/Library/Frameworks/Security.framework/Versions/A/MachServices/SecurityAgent.bundle/Contents/MacOS/SecurityAgent
/System/Library/Frameworks/Security.framework/Versions/A/MachServices/authorizationhost.bundle/Contents/MacOS/authorizationhost
/System/Library/PrivateFrameworks/CoreADI.framework/adid
```

If we go even further and try to search for all binaries on a clean macOS 10.14.5 install that reference `ptrace` we see the following list:

```shell
$ cd /System/Library
$ rg -l -a -M 80 _ptrace 2> /dev/null
Kernels/kernel
QuickTime/QuickTimeComponents.component/Contents/MacOS/QuickTimeComponents
CoreServices/AppleFileServer.app/Contents/MacOS/AppleFileServer
Frameworks/DVDPlayback.framework/Versions/A/DVDPlayback
PrivateFrameworks/CoreADI.framework/Versions/A/adid
PrivateFrameworks/CoreFP.framework/Versions/A/fpsd
PrivateFrameworks/DiskImages.framework/Versions/A/DiskImages
PrivateFrameworks/AnnotationKit.framework/Versions/A/XPCServices/com.apple.AnnotationKit.MigratorService.xpc/Contents/MacOS/com.apple.AnnotationKit.MigratorService
Frameworks/Security.framework/Versions/A/MachServices/SecurityAgent.bundle/Contents/MacOS/SecurityAgent
Frameworks/Security.framework/Versions/A/MachServices/authorizationhost.bundle/Contents/MacOS/authorizationhost
```

I'm not sure if all of these are really using `PT_DENY_ATTACH` or if some are just referencing the text `_ptrace` in some way but they are candidates that could be looked at in the future.

My hope is this post shows a good example of how you can get started with kernel debugging learning about the internals of XNU itself as well as showing that as long as we still have control of the kernel on macOS there's ultimately not much Apple can do to prevent us from inspecting their binaries.
