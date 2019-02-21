---
title: Swift value types vs reference types
categories:
  - Reverse Engineering
tags:
  - Assembly
  - Swift
  - x86
---

A common question that comes up when people start Swift development is what's the difference between a `struct` and a `class`? The standard answer is structs are value types and classes are refernce types. The Swift Programming Language book has a whole [section](https://docs.swift.org/swift-book/LanguageGuide/ClassesAndStructures.html){: target="_blank"} reviewing this concept in more detail. From a reverse engineering perspective I always find it interesting to dive under the hood and see how the compiler actually handles the different concepts from high level languages. This post presents a very simple example of a struct and class in Swift and how the compiler deals with them.

Let's start off with the example Swift code.

```Swift
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

struct CoordStruct {
    let x: Int
    let y: Int
    let z: Int
}

func CoordClassSum(_ c: CoordClass) -> Int {
    return c.x + c.y + c.z;
}

func CoordStructSum(_ c: CoordStruct) -> Int {
    return c.x + c.y + c.z;
}

let c1 = CoordClass(x:0x40, y:0x50, z:0x60)
let c2 = CoordStruct(x:0x10, y:0x20, z:0x30)

print("CoordClassSum  = \(CoordClassSum(c1))\n")
print("CoordStructSum = \(CoordStructSum(c2))\n")
```

You can copy and paste this code into an Xcode playground if you want to play around with it. In both cases we define a coordinate data type composed of a `x`, `y` and `z` value. Then we define a function for the data type that will sum all the values together. We'll start by taking a look at the disassembly of the call to `CoordClassSum` function. 

```asm
000000010000226c 488B3D05645800         mov        rdi, qword [c1]              ; argument #1 for method CoordClassSum, c1
0000000100002273 E848030000             call       CoordClassSum                
```

You can clearly see that the argument to the `CoordClassSum` function is the address of the `c1` object we created. This is where the term "reference type" comes from. We're not passing the `x`, `y` and `z` values directly but rather just a reference to the object that was created. Next lets look at the actual `CoordClassSum` function itself.

```asm
00000001000025c0 55                     push       rbp                          ; CODE XREF=_main+755
00000001000025c1 4889E5                 mov        rbp, rsp
00000001000025c4 48C745F800000000       mov        qword [rbp+var_8], 0x0
00000001000025cc 48897DF8               mov        qword [rbp+var_8], rdi
00000001000025d0 488B4710               mov        rax, qword [rdi+0x10]        ; Set rax = c1.x
00000001000025d4 48034718               add        rax, qword [rdi+0x18]        ; Add c1.y to rax
00000001000025d8 0F90C1                 seto       cl
00000001000025db 48897DF0               mov        qword [rbp+var_10], rdi
00000001000025df 488945E8               mov        qword [rbp+var_18], rax      ; Save the c1.x + c1.y result into a temp var
00000001000025e3 884DE7                 mov        byte [rbp+var_19], cl
00000001000025e6 701E                   jo         loc_100002606                ; Check for overflow

00000001000025e8 488B45E8               mov        rax, qword [rbp+var_18]      ; Set rax to the saved result of c1.x + c1.y
00000001000025ec 488B4DF0               mov        rcx, qword [rbp+var_10]      
00000001000025f0 48034120               add        rax, qword [rcx+0x20]        ; Add c1.z to rax
00000001000025f4 0F90C2                 seto       dl
00000001000025f7 488945D8               mov        qword [rbp+var_28], rax      ; Save final sum onto the stack
00000001000025fb 8855D7                 mov        byte [rbp+var_29], dl
00000001000025fe 7008                   jo         loc_100002608                ; Check for overflow

0000000100002600 488B45D8               mov        rax, qword [rbp+var_28]      ; Return the final value of c1.x + c1.y + c1.z
0000000100002604 5D                     pop        rbp
0000000100002605 C3                     ret
```

In the case of the class you can see how all access to the `x`, `y` and `z` member variables are through the object reference that was originally passed in. The `rdi` register is holding the reference to the object and we access member variables with instructions like `mov rax, qword [rdi+0x10]`. Now lets take a look at the call to the `CoordStructSum`.

```asm
00000001000020ab 488B3DAE655800         mov        rdi, qword [c2.x]            ; argument #1 for method CoordStructSum, x
00000001000020b2 488B35AF655800         mov        rsi, qword [c2.y]            ; argument #2 for method CoordStructSum, y
00000001000020b9 488B15B0655800         mov        rdx, qword [c2.z]            ; argument #3 for method CoordStructSum, z
00000001000020c0 E8AB040000             call       CoordStructSum               
```

Looking at this disassembly you can already see the difference. Even though we have the `CoordStructSum` function defined almost identical to the `CoordClassSum` function, behind the scenes the compiler is going to pass the struct by its individual values. This means that there are really three separate arguements passed to this function. Now lets look at the implementation of `CoordStructSum`.

```asm
0000000100002570 55                     push       rbp                          ; CODE XREF=_main+320
0000000100002571 4889E5                 mov        rbp, rsp
0000000100002574 48897DE8               mov        qword [rbp+var_18], rdi      ; Save c2.x to the stack
0000000100002578 488975F0               mov        qword [rbp+var_10], rsi      ; Save c2.y to the stack
000000010000257c 488955F8               mov        qword [rbp+var_8], rdx       ; Save c2.z to the stack
0000000100002580 4801F7                 add        rdi, rsi                     ; Add c2.x and c2.y
0000000100002583 0F90C0                 seto       al
0000000100002586 48897DE0               mov        qword [rbp+var_20], rdi      ; Save the c2.x + c2.y result into a temp var
000000010000258a 488955D8               mov        qword [rbp+var_28], rdx
000000010000258e 8845D7                 mov        byte [rbp+var_29], al
0000000100002591 701D                   jo         loc_1000025b0                ; Check for overflow

0000000100002593 488B45E0               mov        rax, qword [rbp+var_20]      ; Set rax to the saved result of c2.x + c2.y
0000000100002597 488B4DD8               mov        rcx, qword [rbp+var_28]
000000010000259b 4801C8                 add        rax, rcx                     ; Add c2.z to rax
000000010000259e 0F90C2                 seto       dl
00000001000025a1 488945C8               mov        qword [rbp+var_38], rax      ; Save the final sum onto the stack
00000001000025a5 8855C7                 mov        byte [rbp+var_39], dl
00000001000025a8 7008                   jo         loc_1000025b2                ; Check for overflow

00000001000025aa 488B45C8               mov        rax, qword [rbp+var_38]      ; Return the final value of c2.x + c2.y +c2.z
00000001000025ae 5D                     pop        rbp
00000001000025af C3                     ret
```

So what's the lesson to be learned from all of this? Well from a reverse enginering point of view, when reversing swift code, looking at the functions calls and what's passed in might give some hint to whether the high level language was using a struct or a class to implement the functionality. From a developer view I think it's important to keep in mind that no matter how large your struct is the compiler is going to attempt to copy all the values and pass them in to the function call. In this case, with only three member variables, it doesn't make much of a difference but in a larger struct with variables that are using larger chunks of memory it could.
