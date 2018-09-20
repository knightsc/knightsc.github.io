---
title: Type embedding in Go
categories:
  - Software
tags:
  - Go
---

The Go programming language does not support the concept of inheritance in the traditional object oriented sense. It does allow for type embedding which provides a powerful method for composition. When getting started with Go these patterns are not always obvious. This post provides a short example of how type embedding is used in one of the Go standard library structs.

The [macho](https://golang.org/pkg/debug/macho/) package provides support for reading and manipulating Mach-O binary files. The [Segment](https://github.com/golang/go/blob/master/src/debug/macho/file.go#L60) struct provides a good example of type embedding.

```go
// A Segment represents a Mach-O 32-bit or 64-bit load segment command.
type Segment struct {
	LoadBytes
	SegmentHeader

	// Embed ReaderAt for ReadAt method.
	// Do not embed SectionReader directly
	// to avoid having Read and Seek.
	// If a client wants Read and Seek it must use
	// Open() to avoid fighting over the seek offset
	// with other clients.
	io.ReaderAt
	sr *io.SectionReader
}
```

The comment in the struct indicates that they're not embedding the `SectionReader` directly. This might be confusing at first since a `SectionReader` is obviouslly embedded in the `Segement` struct. The `ReaderAt` interface is embedded without the inclusion of an alias. This means the methods of `ReaderAt` are promoted to the `Segement` struct. In turn, the `Segment` struct can itself be treated directly as a `ReaderAt` instance. The `SectionReader` however is embeded with an alias name. More importantly, the alias name is all lower case, which means it can't be accessed outside of the package. This provides a private `SectionReader` instance that can be used inside of the implementation of the `Segment` struct but no methods are promoted. When using this class outside of the package there's no indication of the `SectionReader` at all.

This short struct does a great job demonstrating not only type embedding but also method promotion and visibility. To read more about type embedding, head over to the [Effective Go](https://golang.org/doc/effective_go.html#embedding) entry on embedding.
