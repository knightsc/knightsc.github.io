---
title: Building XNU 6153.11.26 (almost)
categories:
  - Debugging
tags:
  - C
  - Kernel
  - macOS
---

A couple weeks ago Apple finally releassed the XNU source code for macOS Catalina. It looks like they have now added more of the open source packages needed to build the entire XNU kernel so it's time to update my build instructions.

If you're interested in looking through what's new or changed in the latest XNU code, you can look at a diff of the latest source here:

[https://github.com/knightsc/darwin-xnu/commit/21722f3926534b9dab63b1194b0136ddd6d1fc24](https://github.com/knightsc/darwin-xnu/commit/21722f3926534b9dab63b1194b0136ddd6d1fc24){: target="_blank"}

The first issue that I ran into was compilation issues with `dtrace-338.0.1`. It looks like Apple is now including a handful of llvm header files as part of dtrace in the `include` subfolder. One of those files `include/llvm-Support/PointerLikeTypeTraits.h` still references a file from the `llvmCore` folder structure. So in order to get the dtrace tools to compile I had to generate the missing `DataTypes.h` file and update `PointerLikeTypeTraits.h` to reference it.

With this dtrace header fix in place the build script will run through and generate all of the depedencies needed to try to compile the kernel. Unfortunately it looks like there are some errors compiling the kernel itself that I'm still looking into.

For now if you want to follow along at home, here's a bash script to automate the entire process of building a DEBUG kernel. (Note: It pulls the fixed dtrace header files from gists I've created.):

<script src="https://gist.github.com/knightsc/abbee4ac48b0a43198f5ae39c0f8843f.js"></script>
