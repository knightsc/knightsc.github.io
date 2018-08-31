---
title: macOS and 32-bit applications
categories:
  - Reverse Engineering
tags:
  - macOS
  - MacAdmins
  - osquery
---

https://github.com/knightsc/system_policy

At the 2018 WWDC State of the Union event, following the announcement of macOS Mojave (macOS 10.14) in the keynote earlier in the day, Apple vice president of software Sebastien Marineau revealed Mojave will be "the last release to support 32-bit at all.". Since macOS 10.13.4, Apple has provided the ability to set your machine to 64-bit only mode for testing. For most users this is not a very convenient way to test. As of 10.14 the System Information application has a new "Legacy Software" section that shows you all of the 32-bit applications that have been run on the machine.

This new "Legacy Software" information provides great insight for Mac Admins into what 32-bit applications their users are running so that they can work with vendors to get software updated prior to the release of macOS 10.15 in the fall of 2019.