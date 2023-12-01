---
title: "Demystifying Go runtime: Memory allocator"
date: 2023-11-20T20:00:17+05:30
draft: false
tags: [go,runtime,internals]
---


{{< toc >}}

## Introduction
Memory allocator is responsible for managing heap memory at runtime. Go's allocator
algorithm is based on Google's `tcmalloc` algorithm which is optimised for small
allocations in a multithreaded environment.

Some examples of when memory allocator is invoked
* When non-empty slice is created.
* When `append` is called on the slice `[]T` and capacity needs to be expanded.
* When new elements are added to a `map[K]V`
* When a new goroutine is created with `go f()`
* When a string has to be copied

As you can see this is called quite a lot of times. This also means that one cannot
spin up a new goroutine or simply call `append` as growing a slice calls this subsystem.

I'm using 64-bit **x86-64 (or amd64) on Linux**. So source code explanations assume this environment.

In Go runtime code, the entrypoint of this subsystem is
```go
// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer
```

> NOTE:
> * P = CPU/Processor where it is scheduled
> * M = OS thread which is used to run a goroutine
> * G = Goroutine itself

Depending on the `size` parameter, different allocation algorithms are used. In general, go's
allocator is optimised for small allocations of less than 32KB in concurrent environments.
* **Tiny allocator**: For allocating tiny objects (less than 16B)

## Tiny allocator
Tiny allocator is for values which are less than 16B in size. Several small allocations are
coalesced into a tiny memory block of 16B when possible with the alignment depending on the
size of the object requested. After aligning, if the requested size fits into the tiny slab
then it is allocated in the block assigned to the tiny allocator

### Allocation algorithm

1. Start from `tinyoffset` and align the pointer according to the size requested. Lets call it `alignedOffset`. The alignment rules are specified below.
2. If `alignedOffset+size` fits within the 16B block, then the allocated address is `alignedOffset`

![Tiny allocation algorithm (8B)](/posts/demystifying-go-runtime/pics-memory-allocator/tinyalloc.png)

**Alignment rules**
* If the size is 8B or above, then align it to 8B boundary. As the block size is 16B, if there
  was already a partially filled tiny block with slightly more than 8B left, then this occupies
  the second half of the block
* If the size requested is between 4B to 7B, then it is 4B aligned.
* If the size requested is between 2B or 3B, then it is 2B aligned.
* For 1B request, there is no alignment.


