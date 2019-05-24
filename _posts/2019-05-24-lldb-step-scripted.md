---
title: LLDB step-scripted
categories:
  - Debugging
tags:
  - LLDB
  - macOS
  - Python
---

LLDB has great Python support but it's not always clear what functionality is there or how to use it. While there is documentation on all of the classes available to scripts, some classes and methods are better documented than others. While looking for the best way to trace through some obfuscated assembly recently I came across the `thread step-scripted` command and thought it would be worth writing up a short overview of what it is and how to use it.

If you look at the [LLDB Python Scripting](https://lldb.llvm.org/use/python.html){: target="_blank"} overview or even the documentation of the [LLDB Python classes](https://lldb.llvm.org/python_reference/index.html){: target="_blank"} you won't find any mention of the `thread step-scripted` command. We can get some minimal information from the in app help of LLDB itself:

```shell
(lldb) help thread step-scripted
     Step as instructed by the script class passed in the -C option.

Syntax: thread step-scripted <cmd-options> [<thread-id>]

Command Options Usage:
  thread step-scripted [-a <boolean>] [-A <boolean>] [-c <count>] [-e <linenum>] [-m <run-mode>] [-r <regular-expression>] [-t <function-name>] [<thread-id>]
  thread step-scripted [-C <python-class>] [<thread-id>]

       -C <python-class> ( --python-class <python-class> )
            The name of the class that will manage this step - only supported for Scripted Step.
```

The most complete reference is the following example code from the LLDB source:

[https://github.com/llvm/llvm-project/blob/master/lldb/examples/python/scripted_step.py](https://github.com/llvm/llvm-project/blob/master/lldb/examples/python/scripted_step.py){: target="_blank"} 

In order to use the `thread step-scripted` command we need to define a Python class and pass it to the command. What follows is a short summary of the class structure you need to write and an example of using the command.

# Python class

## \_\_init\_\_(self, thread_plan, dict)

The class you create to handle the scripted step should have a constructor that accepts a [`SBThreadPlan`](https://lldb.llvm.org/python_reference/lldb.SBThreadPlan-class.html){: target="_blank"} and a dictionary. I couldn't figure out what if anything the dictionary should be used for. The thread plan object should be saved as an instance variable as it provides access to the environment you're running in. You can call the `GetThread` method and get a reference to the current [`SBThread`](https://lldb.llvm.org/python_reference/lldb.SBThread-class.html){: target="_blank"} that is being stepped. From the thread you can access the other usual LLDB Python classes like [`SBProcess`](https://lldb.llvm.org/python_reference/lldb.SBProcess-class.html){: target="_blank"}, [`SBTarget`](https://lldb.llvm.org/python_reference/lldb.SBTarget-class.html){: target="_blank"} and [`SBDebugger`](https://lldb.llvm.org/python_reference/lldb.SBDebugger-class.html){: target="_blank"}.

## should_stop(self, event)

This is where most of your logic will go. This method is called after each step. It should return a boolean that indicates if the debugger should stop stepping or not. If you don't implement this method LLDB will default the value to `True`.

## should_step(self)

This method is used to determine if the debugger should keep stepping the target or let it continue to run. Most of the time you will simply want to return `True` from this method. If you are delegating work to a different thread plan then you might want to return `False`. If you don't implement this method LLDB will default to the value `False`.

## explains_stop(self, event)

This method is a little harder to explain. Essentially each thread can be doing different step actions. These actions are referred to as a thread plan and thread plans can be stacked. This means you can have a scripted step that tries to step over 50 instructions and if it hits a breakpoint it will still stop and handle that before continuing on. The thread plans are kept track of on a stack and each one is asked if it can "explain" the stop. If it can then the rest of its code will be called. If this isn't implemented LLDB will default this to return `True`.

## is_stale(self)

I'm not entirely clear on this method. The example class documentation from the LLDB source code stated that this should return a boolean value to indicate if the thread plan is stale. If it is then it will be discarded. The default value LLDB returns if this method isn't implemented is `True`. From what I've seen you probably want to not implement this unless you are delgating actions to another thread plan.

# Example usage

As I mentioned at the beginning of this post, I started looking into the `thread step-scripted` command while reversing some obfuscated assembly code. In this case the assembly code had lots of additional garbage instructions added and as well as numerous indirect `call` statements. I wanted to be able to easily move from `call` to `call` and inspect where the `call` was going and what arguments were being passed in. The sample class below does exactly that. It makes use of the [`SBTarget`](https://lldb.llvm.org/python_reference/lldb.SBTarget-class.html){: target="_blank"} `ReadInstructions` method to disassemble the current instruction and check if the mnemonic has the word `call` in it. If it does then it prints out the argument registers and stops.

<script src="https://gist.github.com/knightsc/4e3b8ab8b8dc3395650467bad6380ee0.js"></script>

Below is an example of using the Python class in a LLDB debugging session:

```shell
$ lldb /System/Library/PrivateFrameworks/CoreADI.framework/adid
(lldb) command script import step.py

(lldb) b 0x1000bd810                              
Breakpoint 1: where = adid`___lldb_unnamed_symbol31$$adid, address = 0x00000001000bd810

(lldb) r
Process 2247 launched: '/System/Library/PrivateFrameworks/CoreADI.framework/adid' (x86_64)
Process 2247 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001000bd810 adid`___lldb_unnamed_symbol31$$adid
adid`___lldb_unnamed_symbol31$$adid:
->  0x1000bd810 <+0>: push   rbp
    0x1000bd811 <+1>: mov    rbp, rsp
    0x1000bd814 <+4>: push   r15
    0x1000bd816 <+6>: push   r14 
Target 0: (adid) stopped.

(lldb) thread step-scripted -C step.Call
0x000000000bd553dc 0x292e90770bd553ac 0x0dc26b67224008e8 0xcccccccccccccccd 0x0000000000000000 0x0000000000000000
Process 2257 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = Python thread plan implemented by class step.Call.
    frame #0: 0x00000001000bd883 adid`___lldb_unnamed_symbol31$$adid + 115
adid`___lldb_unnamed_symbol31$$adid:
->  0x1000bd883 <+115>: call   0x1000a0a40               ; ___lldb_unnamed_symbol17$$adid
    0x1000bd888 <+120>: movzx  edi, al
    0x1000bd88b <+123>: lea    eax, [rdi + 0x1c]
    0x1000bd88e <+126>: lea    r13, [rip + 0xd77eb]
Target 0: (adid) stopped.
```

# Final thoughts

I really like this feature. I think it could be really useful for implementing custom tracing in a Python script that then delegates the stepping logic to a normal `step-over` or `step-out` thread plan. The LLDB instruction disassembly leaves a lot to be desired but you could easily import something like [Capstone](http://www.capstone-engine.org/){: target="_blank"} into the script to get more detailed disassembly information.

One final LLDB related note. Friends don't let friends use AT&T syntax, make sure you add the following to your `~/.lldbinit`. &#128521;

```
settings set target.x86-disassembly-flavor intel
```
