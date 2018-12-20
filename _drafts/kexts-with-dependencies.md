---
title: KEXTs with dependencies
categories:
  - Software
tags:
  - C
  - C++
  - Kernel
  - macOS
---

How to write a kernel module that depends on another kernel module

For example a Base kext that has common datastructures used in multiple other drivers.

Example of defining the dependency as well as getting everything linking and loading correctly.
