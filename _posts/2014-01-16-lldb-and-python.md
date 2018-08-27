---
title: LLDB and Python
categories:
  - Debugging
tags:
  - LLDB
  - macOS
  - Python
---

MacOSX doesn't come with the most recent Python. It's pretty normal to install a newer version either through brew or something else like the anaconda distribution. If you do install a newer version of Python you're likely to encounter problems when trying to access scripting in LLDB.

```shell
$ lldb
(lldb) script help(lldb)
Traceback (most recent call last):
File "~/anaconda/lib/python2.7/site.py", line 557, in
main()
File "~/anaconda/lib/python2.7/site.py", line 539, in main
known_paths = addusersitepackages(known_paths)
File "~/anaconda/lib/python2.7/site.py", line 275, in addusersitepackages
user_site = getusersitepackages()
File "~/anaconda/lib/python2.7/site.py", line 250, in getusersitepackages
user_base = getuserbase() # this will also set USER_BASE
File "~/anaconda/lib/python2.7/site.py", line 240, in getuserbase
USER_BASE = get_config_var('userbase')
File "~/anaconda/lib/python2.7/sysconfig.py", line 516, in get_config_var
return get_config_vars().get(name)
File "~/anaconda/lib/python2.7/sysconfig.py", line 449, in get_config_vars
import re
File "~/anaconda/lib/python2.7/re.py", line 105, in
import sre_compile
File "~/anaconda/lib/python2.7/sre_compile.py", line 14, in
import sre_parse
File "~/anaconda/lib/python2.7/sre_parse.py", line 17, in
from sre_constants import *
File "~/anaconda/lib/python2.7/sre_constants.py", line 18, in
from _sre import MAXREPEAT
ImportError: cannot import name MAXREPEAT
Assertion failed: (err == 0), function ~Mutex, file /SourceCache/lldb/lldb-300.2.51/source/Host/common/Mutex.cpp, line 242.
Abort trap: 6
```

What's happening here is that lldb is loading python scripts from your updated version of python but since lldb itself is linked against an older version things don't match up. If you run the following command you can see what lldb is linked against.

```shell
$ otool -L /Applications/Xcode.app//Contents/SharedFrameworks/LLDB.framework/LLDB
/Applications/Xcode.app//Contents/SharedFrameworks/LLDB.framework/Versions/A/LLDB:
@rpath/LLDB.framework/LLDB (compatibility version 1.0.0, current version 300.2.53)
/System/Library/Frameworks/Carbon.framework/Versions/A/Carbon (compatibility version 2.0.0, current version 157.0.0)
/System/Library/PrivateFrameworks/DebugSymbols.framework/Versions/A/DebugSymbols (compatibility version 1.0.0, current version 106.0.0)
/System/Library/Frameworks/Python.framework/Versions/2.7/Python (compatibility version 2.7.0, current version 2.7.5)
/System/Library/Frameworks/Foundation.framework/Versions/C/Foundation (compatibility version 300.0.0, current version 1052.0.0)
/System/Library/Frameworks/AppKit.framework/Versions/C/AppKit (compatibility version 45.0.0, current version 1247.0.0)
/usr/lib/libxml2.2.dylib (compatibility version 10.0.0, current version 10.9.0)
/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 852.0.0)
/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
/System/Library/Frameworks/Security.framework/Versions/A/Security (compatibility version 1.0.0, current version 55433.0.0)
/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 120.0.0)
/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1197.1.1)
/System/Library/Frameworks/ApplicationServices.framework/Versions/A/ApplicationServices (compatibility version 1.0.0, current version 48.0.0)
/System/Library/Frameworks/CoreServices.framework/Versions/A/CoreServices (compatibility version 1.0.0, current version 59.0.0)
```

Luckily this isn't to hard to change and as long as you're just switching between a minor version it should work okay.

```shell
cd /Applications/Xcode.app//Contents/SharedFrameworks/LLDB.framework/
install_name_tool -change /System/Library/Frameworks/Python.framework/Versions/2.7/Python ~/anaconda/lib/libpython2.7.dylib LLDB
```

After doing that you should be able to work with python scripts in lldb!

```shell
/Applications/Xcode.app//Contents/Developer/usr/bin/lldb
(lldb) script help(lldb)
Help on package lldb:

NAME
lldb - The lldb module contains the public APIs for Python binding.

FILE
/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Versions/A/Resources/Python/lldb/__init__.py

DESCRIPTION
...
```

You'll also want to make sure you're PYTHONHOME is set so lldb can find it and if you want to import the lldb module into a script for use outside of lldb you'll want to update your PYTHONPATH to include the LLDB framework. I stuck the three lines below in my `~/.bash_profile` file.

```shell
PYTHONPATH=$PYTHONPATH:/Applications/Xcode.app/Contents/SharedFrameworks/LLDB.framework/Resources/Python
export PYTHONPATH
export PYTHONHOME=~/anaconda
```
