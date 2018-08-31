---
title: Sandbox Profile Decompiler
categories:
  - Reverse Engineering
tags:
  - Go
  - macOS
  - Security
---

Builds upon this work. Can detect operation names on it's own

https://github.com/malus-security/sandblaster
https://github.com/dionthegod/XNUSandbox
https://github.com/sektioneins/sandbox_toolkit

file-read-data /Library/Application Support/com.apple.TCC

Sandbox: ls(6168) System Policy: deny(1) file-read-data /Library/Application Support/com.apple.TCC
Violation:        System Policy: deny(1) file-read-data /Library/Application Support/com.apple.TCC 

python reverse_sandbox.py -r 11.0.0 -o ../../XNUSandbox/ops.10.13.6.txt -d ../../ "platform_profile.10.13.6"

0000000000027bef

10.14

6580
000000000001e140 s _the_real_platform_profile_data
00000000000246c0 s _the_real_collection_data
0000000000025910 s _path_operations

dd if=Sandbox of=platform_profile skip=123200 count=25984 iflag=skip_bytes,count_bytes

operation_names

0x1b6b8
0x1c25f

0x638
ba7 2983


10.13.6
0x1553
00000000000161b0 s _the_real_platform_profile_data
0000000000017710 s _the_real_collection_data
0000000000018900 s _path_operations

10.12
00000000000136a0
0x13d1

10.11

0x0000000000013fe0
0xa11

10.10
0000000000012430
0x5d1

10.12 collection

0000000000014a80
0xc35

Compiled Sandbox Profile

sb
sbc

Format 1 10.9 10.10, 10.11
Version
RE Offsets
Re Count
Operation nodes

$ xxd 10.10.5/platform_profile.sb.bin 
00000000: 0000 5800 0100 5500 5500 5500 5500 5500  ..X...U.U.U.U.U.
00000010: 5500 5500 5500 5500 5300 5500 5200 5100  U.U.U.U.S.U.R.Q.

$ xxd 10.11.6/platform_profile.sb.bin 
00000000: 0000 8700 0c00 8600 8600 8600 8600 8600  ................
00000010: 8600 8600 8600 8600 7c00 8600 8600 7600  ........|.....v.


Format 2 10.12, 10.3

Version
RE Offsets
Re Count
Global Vars
Global Vars Count

$ xxd 10.12.6/platform_profile.sb.bin 
00000000: 0000 ee00 0900 f100 0000 ed00 ed00 ed00  ................
00000010: eb00 ed00 ed00 ed00 ed00 ed00 ed00 e200  ................

$ xxd 10.13.6/platform_profile.sb.bin 
00000000: 0000 c800 0500 ca00 0000 c700 c700 c700  ................
00000010: c500 c700 c700 c700 c700 c700 c400 c400  ................

Format 3 10.14

Version
RE Offsets
? Something new? 0x2940 any_user_home “unsupported pattern variable”
? Globals? 0x2948
RE Count 0007
Byte newCount 01
Byte globalCount? 1e
Operation nodes

$ xxd 10.14b7/platform_profile.sb.bin 
00000000: 0000 2605 2805 2905 0700 011e 2505 2505  ..&.(.).....%.%.
00000010: 2505 2305 2505 2505 2505 2505 2505 2505  %.#.%.%.%.%.%.%.
00000020: 8704 8704 8104 5f04 a003 e002 de02 da02  ......_.........
00000030: b302 2505 8704 a502 a302 8c02 6404 8704  ..%.........d...
00000040: 2505 db02 4302 4302 6401 5e01 5d01 4302  %...C.C.d.^.].C.

If the third value is greater than the second then we have version 3


(00 0e 0001 002a 002b)
(00 1d 0001 002a 0029)
(00 81 0000 002a 002b)
(01 00 0000 0000 0000)
(01 05 0000 0000 0000)


Entry Offset: 0x002d
Entry Length: 0x0000001f
00000000  00 00 00 03 19 00 2f 15  00 02 2f 02 62 02 69 02  |....../.../.b.i.|
00000010  6e 2f 13 00 02 2f 0a 0b  00 15 00 09 0a 00 00     |n/.../.........|

Version 0x00000003
Length 0x0019

2f 15  00 02 2f 02 62 02 69 02  |....../.../.b.i.|
6e 2f 13 00 02 2f 0a 0b  00 15 00 09 0a 00 00

00: 2f 0015
03: 02 2f “/“
05: 02 62 “b”
07: 02 69 “I”
09: 02 6e “n”
0b: 2f 0013
0e: 02 2f “/“
10: 0a 000b
13: 15 00 - done?
15: 09 .
16: 00a 0000

0000 0000 0003 
0f00 

00: 2f 000b
03: 02 61 “a”
05: 02 62 “b”
07: 02 63 “c”
09: 15 00
0b: 09 
0c: 0a 0000