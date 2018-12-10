---
title: Building XNU 4903.221.2
categories:
  - Debugging
tags:
  - C
  - Kernel
  - macOS
---

Apple finally releassed the XNU source code for macOS Mojave. Oddly enough though it's the source for 10.14.1 with the source for 10.14 still listed as coming soon. Overall the process remains almost identical to building High Sierra. The one signifigant change I noticed was when executing `xcodebuild` commands, I needed to pass the `-UseModernBuildSystem=NO` flag in to get things working properly.

If you're interested in looking through what's new or changed in the latest XNU code, you can look at a diff of the latest source here:

[https://github.com/knightsc/darwin-xnu/commit/61ec6944d0cead45233e18424034f752f3187d21](https://github.com/knightsc/darwin-xnu/commit/61ec6944d0cead45233e18424034f752f3187d21){: target="_blank"}

Finally here's a bash script to automate the entire process of building a DEBUG kernel:

<script src="https://gist.github.com/knightsc/10810d5a0a51d6cdd79daeda99e66daa.js"></script>
