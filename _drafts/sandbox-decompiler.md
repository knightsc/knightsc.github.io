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

/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/buffer.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/condition.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/condition_list.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/context.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/error.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/filter.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/instruction.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/modifier.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/modifiers.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/mutable_buffer.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/operation.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/profile.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/regex.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/source_info.c
/BuildRoot/Library/Caches/com.apple.xbs/Sources/Sandbox/Sandbox-851.201.1/src/compiler/datatypes/textbuf.c

operation
Modifier
Filter
Condition


Apply a filter to actions

Actions are allow or deny

Terminal nodes are actions (allow or deny)
Non terminal nodes are filters


operation_filters
operation_info

0x45f2f - 0x42118

0x3E17

filter_info. NUM_INFO_ENTRIES

modifier_info. modifier != SB_MODIFIER_NONE && modifier < SB_MODIFIER_COUNT

operation_info
0x41bc0- 0x40940
operation_names
0x40938 - 0x404a0 = 0x93
0x1278
 = 32

op < NUM_OPERATIONS

modifier %s does not apply to operation %s"

Arg is condition = is foreign_object?
Arg is operation = is the item an integer number? And is it less than 0x94
Arg is modifier = is the item an integer number and is it less than 0xd

Actions are allow and deny
Not all modifiers can be applied to both actions

___sb_sbpl_parser_init_scheme_block_invoke

Define the allow and deny action methods

What is modifier vs filter
What is condition

Operations

(deny default)

(<action> <operation>)

(deny file-write-data (with no-report))

(<action> <operation> (with <modifier>))

(deny file-read*
       (literal "/private/var/root")
       (literal "/private/var/root/Library/Preferences/.GlobalPreferences.plist")
       (literal "/Library/Preferences/.GlobalPreferences.plist")
       (with no-report))

(<action> <operation>..n <condition>..n (with <modifier>))

0x3fff0 - 0x3f7f0 

This many filters in filter_info 0x40

0x00007fff6ecf7000

sb_filter_accepts_type(condition->filter, condition->value.type)

condition {
    refcnt
    filter
    value {
        type
    }
    match
    unmatch
}

filter < NUM_INFO_ENTRIES && filter_info[filter].name != NULL

typedef struct {
    char *name;
    char *group;
    u8 type
    u8
    u8
    u8
    void *values
} filter_info;

none = 0x0
boolean = 0x1
bit pattern = 0x2
integer = 0x3
string = 0x4
pattern = 0x5
pattern = 0x6
<unknown> = 0x7
path = 0x8
regex = 0x9
network = 0xa

different types of patterns (sb_pattern_is_literal)
