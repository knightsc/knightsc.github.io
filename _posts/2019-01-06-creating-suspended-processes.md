---
title: Creating suspended processes
categories:
  - Malware
tags:
  - C
  - Linux
  - macOS
  - Windows
---

One technique malware uses on Windows to disguise itself is called [process replacement](https://nostarch.com/download/samples/practical-malware-analysis_ch12.pdf){: target="_blank"} or process hollowing. This allows malware to start a well known piece of software like `svchost.exe` in a suspended state, write malicious code into the processes memory and then start the process running. Anyone looking through running processes will simply see a normal `svchost.exe` process running. This has the additional benefit of allowing the malicious code to run with the same privileges as the process it is replacing. You can find a lot of examples of how to create a suspended process on Windows but there doesn't seem to be as many good examples for other platforms. This post will look at Windows, Linux and macOS and how you can create a suspended process on all three operating systems.

We'll start first with Windows since it's the most straight forward. Creating a suspended process is as easy as passing the additional `CREATE_SUSPENDED` flag to the `CreateProcess` API.

```c
STARTUPINFO si = {0};
PROCESS_INFORMATION pi = {0};

si.cb = sizeof(si);
CreateProcess("svchost.exe", NULL, NULL, NULL, FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi));

// Modify suspended process

ResumeThread (pi.hThread);
```

Next up is Linux. On Linux the normal way to create a new process is with the `fork` and `exec` system calls. Like the Windows example above though we want to pause execution of the process we're starting before it runs. After forking in our child process we could have the child send itself a `SIGSTOP` signal but this would pause things before we load the new executable into memory. The only way I could find to do this on linux was to use the `ptrace` APIs. This would enable us to do the normal `fork`/`exec` but receive a signal before things started to run. Here's an example.

```c
void
child()
{
    ptrace(PTRACE_TRACEME, 0, 0, 0);
    execv("./process", argv);
}

void
parent()
{
    int	status;
    waitpid(pid, &status, 0);

    if (WIFSTOPPED(status) && WSTOPSIG(status) == SIGTRAP) {
        // Modify suspended process

        ptrace(PTRACE_CONT, pid, (caddr_t)1, 0);
    }
}

int
main(int argc, char **argv)
{
    pid_t result;

    result = fork();
    switch (result) {
    case 0:
        child();
        break;
    case -1:
        // error
        break;
    default:
        parent();
        break;
    }
}
```

In this example, before the child process calls `exec` it calls the `ptrace` API with the `PTRACE_TRACEME` flag. This allows the parent to trace and control the child. For a more detailed description of how this works, see the excellent [blog post](http://system.joekain.com/2015/06/08/debugger.html){: target="_blank"} by Joseph Kain.

Now on to macOS. Since macOS is unix based and does have the `ptrace` APIS the example above will work on macOS. You just have to modify the flags to be `PT_TRACE_ME` and `PT_CONTINUE` respecitvely. With the goal of simply creating a suspended process however there's an even easier way to accomplish this on macOS. Apple has a handful of macOS specific posix spawn attribute flags.

| **POSIX_SPAWN_SETEXEC**         | *Apple Extension*: If this bit is set, rather than returning to the caller, posix_spawn(2) and posix_spawnp(2) will behave as a more featureful execve(2). |
| **POSIX_SPAWN_START_SUSPENDED** | *Apple Extension*: If this bit is set, then the child process will be created as if it immediately received a SIGSTOP signal, permitting debuggers, profilers, and other programs to manipulate the process before it begins execution in user space.  This permits, for example, obtaining exact instruction counts, or debugging very early in dyld(1).  To resume the child process, it must be sent a SIGCONT signal. |
| **POSIX_SPAWN_CLOEXEC_DEFAULT** | *Apple Extension*: If this bit is set, then only file descriptors explicitly described by the file_actions argument are available in the spawned process; all of the other file descriptors are automatically closed in the spawned process. |

So we should be able to simply use the `posix_spawn` function passing in the `POSIX_SPAWN_START_SUSPENDED` flag. Here's an example of what that would look like.

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <spawn.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <signal.h>

int
main(int argc, char **argv)
{
    pid_t pid;
    int status;
    posix_spawnattr_t attr;

    status = posix_spawnattr_init(&attr);
    if (status != 0) { 
        perror("can't init spawnattr"); 
        exit(status); 
    }

    status = posix_spawnattr_setflags(&attr, POSIX_SPAWN_START_SUSPENDED);
    if (status != 0) { 
        perror("can't set flags"); 
        exit(status); 
    }

    status = posix_spawn(&pid, "./hello", NULL, &attr, NULL, NULL);
    if (status != 0) {
        printf("posix_spawn: %s\n", strerror(status));
        exit(status);
    }

    printf("Child pid: %i\n", pid);
            
    sleep(10);
    
    kill(pid, SIGCONT);

    return 0;
}
```

In this case after the process has been started in the suspended state we simply send it the `SIGCONT` signal to get it to resume. macOS does have an undocumented `pid_resume` system call but it's not clear if there's restrictions on who can call it or not.

Hopefully these three examples were helpful. I think it's useful to study malicious techniques on one platform and see how they might be adapted to another. I would expect malware authors to be doing the same thing as they continue to create more malicious software for platforms that aren't Windows.
