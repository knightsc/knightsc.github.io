---
title: Swift type metadata
categories:
  - Reverse Engineering
tags:
  - Swift
---

Apple does have some documentation on Swift metadata but it's not entirely clear

https://github.com/apple/swift/blob/master/docs/ABI/TypeMetadata.rst

Some of the documentation explicitly states that it's out of date

> The Swift runtime keeps a metadata record for every type used in a program, including every instantiation of generic types. These metadata records can be used by (TODO: reflection and) debugger tools to discover information about types.

Robust metadata is great for reverse engineering but sometimes you need to understand what the metadata is to make it useful. This article covers the Swift 5 and up metadata system from the perspective of a reverse engineer and tries to provide descriptions that you could use to write your own tools to read and use the information.

	Mem: 0x100002986-0x1000029d0		__TEXT.__swift5_typeref
	Mem: 0x1000029d0-0x100002a57		__TEXT.__cstring	(C-String Literals)
	Mem: 0x100002a58-0x100002d2a		__TEXT.__const
	Mem: 0x100002d2c-0x100002e60		__TEXT.__swift5_fieldmd
	Mem: 0x100002e60-0x100002ebc		__TEXT.__swift5_reflstr
	Mem: 0x100002ebc-0x100002ec4		__TEXT.__swift5_proto
	Mem: 0x100002ec4-0x100002ee4		__TEXT.__swift5_types

## __swift5_types

Array of 32-bit unsigned relative offsets to nominal type descriptors

Flags
Parent
Name
Accessor function (this function points to full metadata for reflection)

## __swift5_proto

Array of 32-bit unsigned relative offsets to protocol conformance descriptors

## __swift5_fieldmd

Reflection metadata for field descriptor? What does this mean? Where are field descriptors stored or does this reference a field descriptor. Referenced from nominal type descriptors

## __swift5_reflstr

Reflection data for names of properties

## __swift5_typeref

? Not sure

## __swift5_assocty

## __swift5_builtin

## __swift5_capture

## __swift5_replace

## __swift5_replac2

## __swift5_protos

What is this verse proto? Maybe this is nominal type descriptors versus conformances?

## Other

What are the symbolic references