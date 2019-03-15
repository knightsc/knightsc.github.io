---
title: Detecting task modifications
categories:
  - Software
tags:
  - Go
  - macOS
  - MacAdmins
  - osquery
---

Talk about crash reportedr it actually reports this information

```
External Modification Summary:
  Calls made by other processes targeting this process:
    task_for_pid: 306
    thread_create: 0
    thread_set_state: 16
  Calls made by this process:
    task_for_pid: 0
    thread_create: 0
    thread_set_state: 0
  Calls made by all processes on this machine:
    task_for_pid: 14911679
    thread_create: 55
    thread_set_state: 275163
```

So is there a way that we can inspect and detect this condition ourself? Yes!!

Talk a little about the task struct in XNU

Talk about how extmod info is populated

Show the example

Look for it in msgtracer logs


<script src="https://gist.github.com/knightsc/06a1b74b779690e8e491c21a3883c7a7.js"></script>

Could be put into an osquery table <or create an osquery table example>
