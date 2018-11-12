---
title: Swift Value Types vs Reference Types
categories:
  - Software
tags:
  - Swift
---

[https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html)  


```Swift
struct CoordStruct {
    let x: Int
    let y: Int
    let z: Int
}

class CoordClass {
    let x: Int
    let y: Int
    let z: Int
    
    init(x: Int, y: Int, z: Int) {
        self.x = x
        self.y = y
        self.z = z
    }
}

func CoordStructSum(_ c: CoordStruct) -> Int {
    return c.x + c.y + c.z;
}

func CoordClassSum(_ c: CoordClass) -> Int {
    return c.x + c.y + c.z;
}

let c1 = CoordStruct(x:0x10, y:0x20, z:0x30)
let c2 = CoordClass(x:0x40, y:0x50, z:0x60)

print("CoordSum      = \(CoordStructSum(c1))\n")
print("CoordClassSum = \(CoordClassSum(c2))\n")
```



```asm
00000001000020ab 488B3DAE655800         mov        rdi, qword [x]               ; argument #1 for method CoordSum, x
00000001000020b2 488B35AF655800         mov        rsi, qword [y]               ; argument #2 for method CoordSum, y
00000001000020b9 488B15B0655800         mov        rdx, qword [z]               ; argument #3 for method CoordSum, z
00000001000020c0 E8AB040000             call       CoordSum                     ; CoordSum
```

```asm
0000000100002570 55                     push       rbp                          ; CODE XREF=_main+320
0000000100002571 4889E5                 mov        rbp, rsp
0000000100002574 48897DE8               mov        qword [rbp+var_18], rdi
0000000100002578 488975F0               mov        qword [rbp+var_10], rsi
000000010000257c 488955F8               mov        qword [rbp+var_8], rdx
0000000100002580 4801F7                 add        rdi, rsi
0000000100002583 0F90C0                 seto       al
0000000100002586 48897DE0               mov        qword [rbp+var_20], rdi
000000010000258a 488955D8               mov        qword [rbp+var_28], rdx
000000010000258e 8845D7                 mov        byte [rbp+var_29], al
0000000100002591 701D                   jo         loc_1000025b0

0000000100002593 488B45E0               mov        rax, qword [rbp+var_20]
0000000100002597 488B4DD8               mov        rcx, qword [rbp+var_28]
000000010000259b 4801C8                 add        rax, rcx
000000010000259e 0F90C2                 seto       dl
00000001000025a1 488945C8               mov        qword [rbp+var_38], rax
00000001000025a5 8855C7                 mov        byte [rbp+var_39], dl
00000001000025a8 7008                   jo         loc_1000025b2

00000001000025aa 488B45C8               mov        rax, qword [rbp+var_38]
00000001000025ae 5D                     pop        rbp
00000001000025af C3                     ret
```

```asm
000000010000226c 488B3D05645800         mov        rdi, qword [c2]              ; argument #1 for method CoordClassSum, c2
0000000100002273 E848030000             call       CoordClassSum                ; CoordClassSum
```

```asm
00000001000025c0 55                     push       rbp                          ; CODE XREF=_main+755
00000001000025c1 4889E5                 mov        rbp, rsp
00000001000025c4 48C745F800000000       mov        qword [rbp+var_8], 0x0
00000001000025cc 48897DF8               mov        qword [rbp+var_8], rdi
00000001000025d0 488B4710               mov        rax, qword [rdi+0x10]
00000001000025d4 48034718               add        rax, qword [rdi+0x18]
00000001000025d8 0F90C1                 seto       cl
00000001000025db 48897DF0               mov        qword [rbp+var_10], rdi
00000001000025df 488945E8               mov        qword [rbp+var_18], rax
00000001000025e3 884DE7                 mov        byte [rbp+var_19], cl
00000001000025e6 701E                   jo         loc_100002606

00000001000025e8 488B45E8               mov        rax, qword [rbp+var_18]
00000001000025ec 488B4DF0               mov        rcx, qword [rbp+var_10]
00000001000025f0 48034120               add        rax, qword [rcx+0x20]
00000001000025f4 0F90C2                 seto       dl
00000001000025f7 488945D8               mov        qword [rbp+var_28], rax
00000001000025fb 8855D7                 mov        byte [rbp+var_29], dl
00000001000025fe 7008                   jo         loc_100002608

0000000100002600 488B45D8               mov        rax, qword [rbp+var_28]
0000000100002604 5D                     pop        rbp
0000000100002605 C3                     ret
```