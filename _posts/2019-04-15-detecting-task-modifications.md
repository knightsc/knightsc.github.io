---
title: Detecting task modifications
categories:
  - Reverse Engineering
tags:
  - Kernel
  - macOS
  - Security
---

In the [previous post]({% post_url 2019-03-15-code-injection-on-macos %}){: target="_blank"} we looked at different ways to inject code into tasks on macOS. The goal being to create increased awareness of the type of methods attackers writing malicious code on macOS might use. In this post I wanted to focus in on the same issue of code injection but from a defenders point of view. 

While starting this investigation I happened to notice the following information on some of the crash reports that macOS generates.

```
External Modification Summary:
  Calls made by other processes targeting this process:
    task_for_pid: 2
    thread_create: 0
    thread_set_state: 0
  Calls made by this process:
    task_for_pid: 0
    thread_create: 0
    thread_set_state: 0
  Calls made by all processes on this machine:
    task_for_pid: 11110019
    thread_create: 2
    thread_set_state: 524
```

The `Calls made by other processes targeting this process` looked interesting and seemed potentially related to what I was trying to figure out. If we could get this same information then we could potentially detect when an outside process has attempted to inject or hijack a thread. I started by searching my system to see what generated this section of the crash report. I used ripgrep to search for anything mentioning the text above:

```shell
$ rg -a -l -M 80 "Calls made by other processes targeting this process" 2>/dev/null
/System/Library/CoreServices/ReportCrash
```

Opening `ReportCrash` in [Hopper](https://www.hopperapp.com/){: target="_blank"} and locating the reference to the string took me to the `CrashReport` Objective-C class which has a method called `_extractExternalMods`. This method starts out and does the following:

```asm
0000000100009a85 lea        rcx, qword [rbp+var_74]
0000000100009a89 mov        dword [rcx], 0x10            ; argument "task_info_outCnt" for method imp___stubs__task_info
0000000100009a8f mov        rax, qword [objc_ivar_offset_CrashReport__task] ; objc_ivar_offset_CrashReport__task
0000000100009a96 mov        edi, dword [rdi+rax]         ; argument "target_task" for method imp___stubs__task_info
0000000100009a99 lea        rdx, qword [rbp+var_70]      ; argument "task_info_out" for method imp___stubs__task_info
0000000100009a9d mov        esi, 0x13                    ; argument "flavor" for method imp___stubs__task_info
0000000100009aa2 call       imp___stubs__task_info       ; task_info
0000000100009aa7 test       eax, eax
0000000100009aa9 jne        loc_100009d4f
```

Looking at the [XNU source code](https://github.com/apple/darwin-xnu/blob/master/osfmk/mach/task_info.h#L315){: target="_blank"} to see what `flavor` 0x13 is, we see that this is the `TASK_EXTMOD_INFO` constant. It looks like this information has been available since macOS 10.7 in xnu-1699.22.73. Translating this code to C we see that it's doing the following:

```c
struct task_extmod_info *info;
mach_msg_type_number_t count = TASK_EXTMOD_INFO_COUNT;
kern_return_t kr;

kr = task_info(self.task, TASK_EXTMOD_INFO, (task_info_t)info, &count);
```

If we can get a reference to a task port we can call `task_info` passing in `TASK_EXTMOD_INFO` and get back out the following information:

```c
struct vm_extmod_statistics {
  int64_t	task_for_pid_count;		/* # of times task port was looked up */
  int64_t task_for_pid_caller_count;		/* # of times this task called task_for_pid */
  int64_t	thread_creation_count;		/* # of threads created in task */
  int64_t	thread_creation_caller_count;	/* # of threads created by task */
  int64_t	thread_set_state_count;		/* # of register state sets in task */
  int64_t	thread_set_state_caller_count;	/* # of register state sets by task */
} __attribute__((aligned(8)));

typedef struct vm_extmod_statistics vm_extmod_statistics_data_t;

struct task_extmod_info {
  unsigned char	task_uuid[16];
  vm_extmod_statistics_data_t extmod_statistics;
};
```

If `thread_creation_count` or `thread_set_state_count` is greater than zero it means an external user task has modified the task we're inspecting. 

Since we need to pass a task reference into `task_info` you might be thinking that our own program won't be able to call into this function. (Apple goes to great lengths to lock down `task_for_pid`). If you look at how the MIG call for `task_info` works you'll notice the following logic:

```c
kern_return_t
task_info_from_user(
  mach_port_t		task_port,
  task_flavor_t		flavor,
  task_info_t		task_info_out,
  mach_msg_type_number_t	*task_info_count)
{
  task_t task;
  kern_return_t ret;

  if (flavor == TASK_DYLD_INFO)
    task = convert_port_to_task(task_port);
  else
    task = convert_port_to_task_name(task_port);

  ret = task_info(task, flavor, task_info_out, task_info_count);

  task_deallocate(task);

  return ret;
}
```

Unless we're passing in a `flavor` of `TASK_DYLD_INFO` we don't need a full task port. We only need to have a task name port. As a regular user we can retrieve a task name port using `task_name_for_pid`. There is one drawback with `task_name_for_pid` though, it requires being called from a privileged process or a processes with the same user ID. So if we put all these pieces together along with a call to `proc_listallpids` we come up with the following code:

<script src="https://gist.github.com/knightsc/06a1b74b779690e8e491c21a3883c7a7.js"></script>

If you call this code as a regular user it will inspect all processes running as that user. If you call the code with sudo then it will inspect all processes running on the machine. Here's an example of injecting a dylib into Slack and then using the code above to detect the modifications.

```shell
$ sudo ./inject 1634 libinjectme.dylib 
Allocated remote stack @0x10b1e0000
pthread_create_from_mach_thread @7fff5df23d54
dlopen @7fff5dd1dc7b
Remote Stack 64  0x10b1e8000, Remote code is 0x10a39c000
Stub thread finished

$ ./psx
PID: 1634
Name: Slack
External Modification Summary:
  Calls made by other processes targeting this process:
    task_for_pid: 397
    thread_create: 1
    thread_set_state: 0
```

Alternatively, looking through the XNU source code, creating remote threads in another process does get logged. You can search the logs for thread injection with the following command:

```shell
$ syslog -F raw -d /var/log/DiagnosticMessages | grep "com.apple.kernel.external_modification"

NOTE:  Most system logs have moved to a new logging system.  See log(1) for more information.
[ASLMessageID 56229] [Time 1555339907] [TimeNanoSec 0] [Level 7] [PID 0] [UID 0] [GID 0] [ReadUID 0] [ReadGID 80] [Host HOST1] [Sender kernel] [Facility messagetracer] [com.apple.message.domain com.apple.kernel.external_modification] [com.apple.message.signature inject(76017065-CAFD-39C7-8629-669E35802D89)] [com.apple.message.signature2 Slack(22CA3711-E38C-376F-A0BC-DCD5F47D0BBC)] [com.apple.message.result noop] [com.apple.message.summarize YES]
```

I think the APIs and approach above are a fairly reliable way to check macOS processes for external modifications. In general the only legitimate things I've found doing thread injection or thread hijacking are a few Apple development binaries or crash reporting tools. When I look at this from the perspective of managing mac computers, it's definitely an indicator that I plan to look into including in future reporting. I don't expect that it would alert very often, but if it does, it's definitely something that I would want to investigate further.
