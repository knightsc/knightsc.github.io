---
title: Swift metadata
categories:
  - Reverse Engineering
tags:
  - Swift
---

When reverse engineering macOS binaries that are written in Objective-C, [class-dump](https://github.com/nygard/class-dump/){: target="_blank"} is a common tool used to extract Objective-C declarations from the runtime information stored in the Mach-O files. With Swift binaries, since there is Objective-C compatability, sometimes you can extract declarations using [class-dump](https://github.com/nygard/class-dump/){: target="_blank"} but not always. Swift has a rich set of type metadata itself but the [documentation](https://github.com/apple/swift/blob/master/docs/ABI/TypeMetadata.rst){: target="_blank"} is not up to date. With Swift 5 bringing [ABI stability](https://swift.org/blog/abi-stability-and-more/){: target="_blank"} I thought it would be interesting to take a look at the type of metadata availble in Swift binaries.

From the [documentation](https://github.com/apple/swift/blob/master/docs/ABI/TypeMetadata.rst){: target="_blank"} we see the following overview of Swift type metadata:

> The Swift runtime keeps a metadata record for every type used in a program, including every instantiation of generic types. These metadata records can be used by (TODO: reflection and) debugger tools to discover information about types.

To start off with lets start with finding out where the Swift metadata is stored in a Mach-O file. `jtool -l` or `otool -l` will work in this case.

```shell
$ jtool2 -l /Applications/Stocks.app/Contents/MacOS/Stocks
LC 00: LC_SEGMENT_64              Mem: 0x000000000-0x100000000    __PAGEZERO
LC 01: LC_SEGMENT_64              Mem: 0x100000000-0x100028000    __TEXT
    Mem: 0x100002ca0-0x100020304        __TEXT.__text    (Normal)
    Mem: 0x100020304-0x100020a90        __TEXT.__stubs    (Symbol Stubs)
    Mem: 0x100020a90-0x100021734        __TEXT.__stub_helper    (Normal)
    Mem: 0x100021740-0x100022c28        __TEXT.__const
    Mem: 0x100022c30-0x100024dc5        __TEXT.__cstring    (C-String Literals)
    Mem: 0x100024dc5-0x100026630        __TEXT.__objc_methname    (C-String Literals)
    Mem: 0x100026630-0x100026d54        __TEXT.__swift5_typeref
    Mem: 0x100026d60-0x100027061        __TEXT.__swift5_reflstr
    Mem: 0x100027064-0x1000274cc        __TEXT.__swift5_fieldmd
    Mem: 0x1000274cc-0x100027608        __TEXT.__swift5_capture
    Mem: 0x100027608-0x100027668        __TEXT.__swift5_assocty
    Mem: 0x100027668-0x1000276f8        __TEXT.__swift5_proto
    Mem: 0x1000276f8-0x10002776c        __TEXT.__swift5_types
    Mem: 0x10002776c-0x100027794        __TEXT.__swift5_builtin
    Mem: 0x100027794-0x1000277ac        __TEXT.__swift5_protos
    Mem: 0x1000277ac-0x100027bb0        __TEXT.__unwind_info
    Mem: 0x100027bb0-0x100027ff8        __TEXT.__eh_frame
```

We can see that in the `__TEXT` segment of the binary there are a handful of sections that start with the `__swift5` prefix. With the documentation out of date the best way to determine what goes in these sections is to look directly in the Swift code. Here are some of the most helpful places in code I've found:

* [https://github.com/apple/swift/blob/master/include/swift/ABI/Metadata.h](https://github.com/apple/swift/blob/master/include/swift/ABI/Metadata.h){: target="_blank"}
* [https://github.com/apple/swift/blob/master/include/swift/Reflection/Records.h](https://github.com/apple/swift/blob/master/include/swift/Reflection/Records.h){: target="_blank"}
* [https://github.com/apple/swift/blob/master/lib/IRGen/GenDecl.cpp](https://github.com/apple/swift/blob/master/lib/IRGen/GenDecl.cpp){: target="_blank"}
* [https://github.com/apple/swift/blob/master/lib/IRGen/GenMeta.cpp](https://github.com/apple/swift/blob/master/lib/IRGen/GenMeta.cpp){: target="_blank"}

These are just a few of the example files I found useful while looking into Swift metadata. In general the useful code breaks down into ABI code, reflection code and LLVM IR generation code. The metadata in the LLVM IR code is nicely labelled and is the same as what gets emitted in the final compiled binary. If you have a test `.swift` file you can generate the IR yourself for inspection using the following command:

```shell
swiftc -emit-ir -Xfrontend -disable-llvm-optzns -O test.swift > test.ir
```

What follows is a high level description of all the Swift 5 sections that can show up in a Swift binary.

# __TEXT.__swift5_protos

This section contains an array of 32-bit signed integers. Each integer is a relative offset that points to a protocol descriptor in the `__TEXT.__const` section.

A protocol descriptor has the following format:

```go
type ProtocolDescriptor struct {
    Flags                      uint32
    Parent                     int32
    Name                       int32
    NumRequirementsInSignature uint32
    NumRequirements            uint32
    AssociatedTypeNames        int32
}
```

# __TEXT.__swift5_proto

This section contains an array of 32-bit signed integers. Each integer is a relative offset that points to a protocol conformance descriptor in the `__TEXT.__const` section.

A protocol conformance descriptor has the following format:

```go
type ProtocolConformanceDescriptor struct {
    ProtocolDescriptor    int32 
    NominalTypeDescriptor int32
    ProtocolWitnessTable  int32
    ConformanceFlags      uint32
}
```

# __TEXT.__swift5_types

This section contains an array of 32-bit signed integers. Each integer is a relative offset that points to a nominal type descriptor in the `__TEXT.__const` section.

A nominal type descriptor can take different formats based on what type it represents. There are different structures for `Enum`, `Struct` and `Class` types. They have the following format:

```go
type EnumDescriptor struct {
    Flags                               uint32
    Parent                              int32
    Name                                int32
    AccessFunction                      int32
    FieldDescriptor                     int32
    NumPayloadCasesAndPayloadSizeOffset uint32
    NumEmptyCases                       uint32
}

type StructDescriptor struct {
    Flags                   uint32
    Parent                  int32
    Name                    int32
    AccessFunction          int32
    FieldDescriptor         int32
    NumFields               uint32
    FieldOffsetVectorOffset uint32
}

type ClassDescriptor struct {
    Flags                       uint32
    Parent                      int32
    Name                        int32
    AccessFunction              int32
    FieldDescriptor             int32
    SuperclassType              int32
    MetadataNegativeSizeInWords uint32
    MetadataPositiveSizeInWords uint32
    NumImmediateMembers         uint32
    NumFields                   uint32
}
```

# __TEXT.__const

This section is not Swift specific. It contains descriptors referenced from other sections. Here are some of the various structures you'll see in this section:

* Protocol conformance descriptor
* Module descriptor
* Protocol descriptor
* Nominal type descriptors
* Direct field offsets
* Method descriptors

# __TEXT.__swift5_fieldmd

This section contains an array of field descriptors. A field descriptor contains a collection of field records for a single class, struct or enum declaration. Each field descriptor can be a different length depending on how many field records the type contains.

A field descriptor has the following format:

```go
type FieldRecord struct {
    Flags           uint32
    MangledTypeName int32
    FieldName       int32
}

type FieldDescriptor struct {
    MangledTypeName int32
    Superclass      int32
    Kind            uint16
    FieldRecordSize uint16
    NumFields       uint32
    FieldRecords    []FieldRecord
}
```

# __TEXT.__swift5_assocty

This section contains an array of associated type descriptors. An associated type descriptor contains a collection of associated type records for a conformance. An associated type records describe the mapping from an associated type to the type witness of a conformance.

An associated type descriptor has the following format:

```go
type AssociatedTypeRecord struct {
    Name                int32
    SubstitutedTypeName int32
}

type AssociatedTypeDescriptor struct {
    ConformingTypeName       int32
    ProtocolTypeName         int32
    NumAssociatedTypes       uint32
    AssociatedTypeRecordSize uint32
    AssociatedTypeRecords    []AssociatedTypeRecord
}
```

# __TEXT.__swift5_builtin

This section contains an array of builtin type descriptors. A builtin type descriptor describes the basic layout information about any builtin types referenced from other sections.

A builtin type descriptor has the following format:

```go
type BuiltinTypeDescriptor struct {
    TypeName            int32
    Size                uint32
    AlignmentAndFlags   uint32
    Stride              uint32
    NumExtraInhabitants uint32
}
```

# __TEXT.__swift5_capture

Capture descriptors describe the layout of a closure context object. Unlike nominal types, the generic substitutions for a closure context come from the object, and not the metadata.

A capture descriptor has the following format:

```go
type CaptureTypeRecord struct {
    MangledTypeName int32
}

type MetadataSourceRecord struct {
    MangledTypeName       int32
    MangledMetadataSource int32
}

type CaptureDescriptor struct {
    NumCaptureTypes       uint32
    NumMetadataSources    uint32
    NumBindings           uint32
    CaptureTypeRecords    []CaptureTypeRecord
    MetadataSourceRecords []MetadataSourceRecord
}
```

# __TEXT.__swift5_typeref

This section contains a list of mangled type names that are referenced from other sections. This is essentially all the different types that are used in the application. The Swift docs and code are the best places to find out more information about mangled type names.

* [https://github.com/apple/swift/blob/master/docs/ABI/Mangling.rst](https://github.com/apple/swift/blob/master/docs/ABI/Mangling.rst){: target="_blank"}
* [https://github.com/apple/swift/blob/master/lib/Demangling/Demangler.cpp](https://github.com/apple/swift/blob/master/lib/Demangling/Demangler.cpp){: target="_blank"}

# __TEXT.__swift5_reflstr

This section contains an array of C strings. The strings are field names for the properties of the metadata defined in other sections.

# __TEXT.__swift5_replace

This section contains dynamic replacement information. This is essentially the Swift equivalent of Objective-C method swizzling. You can read more about this Swift feature [here](https://forums.swift.org/t/dynamic-method-replacement/16619){: target="_blank"}.

The dynamic replacement information has the following format:

```go
type Replacement struct {
    ReplacedFunctionKey int32
    NewFunction         int32
    Replacement         int32
    Flags               uint32
}

type ReplacementScope struct {
    Flags uint32
    NumReplacements uint32
    
}

type AutomaticReplacements struct {
    Flags            uint32
    NumReplacements  uint32 // hard coded to 1
    Replacements     int32
}
```

# __TEXT.__swift5_replac2

This section contains dynamica replacement information for opaque types. It's not clear why this additional section was created instead of `__swift5_replace` but you can see the original pull request that implemented it [here](https://github.com/apple/swift/pull/24781){: target="_blank"}.

The dynamic replacement information for opaque types has the following format:

```go
type Replacement struct {
    Original    int32
    Replacement int32
}

type AutomaticReplacementsSome struct {
    Flags uint32
    NumReplacements uint32
    Replacements    []Replacement
}
```

# __DATA.__const

This section is not Swift specific. It contains things like value witness tables and full type metadata. Presumably some of the metadata is placed here so it can be modified at runtime.

# Wrapping up

My hope is the information above helps make it more clear what type of extra information is stored in Swift binaries. Just like was mentioned in the official metadata documentation this information can be used for writing debug tools. It should be possible to use this information to create a [class-dump](https://github.com/nygard/class-dump/){: target="_blank"} like tool for Swift binaries.
