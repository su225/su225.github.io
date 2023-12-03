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

Think of any "memory manager" as owning the whole heap region in the process address space.
It "hands out" pieces of memory to various threads/goroutines asking for memory at runtime.
In other words, these threads **borrow** memory from the heap memory manager which in turn
**borrows** real, physical memory from the Operating System. Freeing (whether manually or
by a Garbage Collector) is the opposite. The "borrowed memory" is returned to the memory
manager. Similarly, the "heap memory manager" might also return it to the Operating System.

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
* Allocating from the per-M cached, span called `mspan` (up to 32KB)
* Allocating directly from the system above that.

High-level overview of the Go's memory allocation algorithm
{{< mermaid >}}
flowchart TD
    A[Christmas] -->|Get money| B(Go shopping)
    B --> C{Let me think}
    C -->|One| D[Laptop]
    C -->|Two| E[iPhone]
    C -->|Three| F[fa:fa-car Car]
  
{{< /mermaid >}}

## Tiny allocator
Tiny allocator is for values which are less than 16B in size. Several small allocations are
coalesced into a tiny memory block of 16B when possible with the alignment depending on the
size of the object requested. After aligning, if the requested size fits into the tiny slab
then it is allocated in the block assigned to the tiny allocator.

Tiny allocator is only used when the type is not a pointer (called `noscan` in code). This way,
the Garbage Collector can see the whole 16B as one block and free accordingly without having
to traverse through the block searching for more references from it.

### Allocation algorithm

TL;DR - **bump the pointer (with some alignment rules)**

1. Start from `tinyoffset` and align the pointer according to the size requested. Lets call it `alignedOffset`. The alignment rules are specified below.
2. If `alignedOffset+size` fits within the 16B block, then the allocated address is `alignedOffset`
3. If it does not fit, then allocate a new 16B block from the `mspan`, set `tinyoffset=0` and allocate.

![Tiny allocation algorithm (8B)](/posts/demystifying-go-runtime/pics-memory-allocator/tinyalloc.png)

**Alignment rules**
* If the size is 8B or above, then align it to 8B boundary. As the block size is 16B, if there
  was already a partially filled tiny block with slightly more than 8B left, then this occupies
  the second half of the block
* If the size requested is between 4B to 7B, then it is 4B aligned.
* If the size requested is between 2B or 3B, then it is 2B aligned.
* For 1B request, there is no alignment.

## Implementation details
Exciting stuff! Lets dive into the source code of Go runtime which is also written in Go! I'm using
Linux with amd64 architecture (64-bit) and I'll focus on that.

### Data structures
The memory manager manages the heap and hence its data structures should live outside the heap. Some
important data structures of the memory allocator are as follows
* `mheap`: Central to the Go's memory management. It is a global data structure which contains everything
* `mcentral`: Contains `mspan` for small size classes (upto 32KB). This is one per `mheap`. Hence,
  accessing it requires holding the lock on `mheap`. Hence, each OS thread has an `mcache` where the
  `mspan` for each size class is cached and allocated.
* `mcache`: Per-M cache containing `mspan` of various size classes for small allocations. Allocating from
  an `mcache` does not require holding `mheap` lock and hence it is fast and reduces contention. However,
  when there is no `mspan` to satisfy an allocation request, it requests `mcentral` to refill the cache. 

It also has dynamic data structures and they are managed by internal allocators which are quite simple to
understand and implement.
* `persistentalloc`: Get blocks of memory from the underlying OS and this is **never freed**.
* `fixalloc`: Gets blocks of memory from `persistentalloc` and carves it up into fixed size objects.

I mentioned earlier that allocating slice `[]T` might call the memory manager (`mallocgc`) as part of
expanding the slice. However, `mheap` also has **off-heap** slice data structures and the memory manager 
**CANNOT** call `append` because it might be a recursive call back to itself. The memory backing these 
slices are allocated from `persistentalloc` as it is **off-heap**. And `map[K]V` are not used at all
because it is much more complex than the slice type.

#### `mheap`
This is the global data structure used by the allocator.
```go
var mheap_ mheap
```
All other allocations mentioned earlier `mspan`, `mcache` etc are registered here. Naturally, garbage
collector also starts from here.

#### `mcentral`

#### `mcache`

#### `mspan`
An `mspan` consists of `nelem` number of equally-sized objects. The size of the object depends on
the size class of the span. It is indicated by the `spanclass` member. This is used for small objects
up to the size of 32KB only.

#### `heapArena`

### Memory-management internal allocators

#### `persistentalloc`

#### `fixalloc`

