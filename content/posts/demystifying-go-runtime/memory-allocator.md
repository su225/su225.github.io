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
I also assume we are not using Cgo.

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

## Implementation details
Exciting stuff! Lets dive into the source code of Go runtime which is also written in Go! I'm using
Linux with amd64 architecture (64-bit) and I'll focus on that.

The memory manager manages the heap and hence its data structures should live outside the heap.
* `mheap`: Central data structure which contains all the information required to manage the heap and GC.
* `mcache`: Per-M allocator for allocating objects of small size up to 32KB.
* `tinyalloc`: Non-pointer types upto 16B are allocated inside a 16B block inside `mcache`
* `mspan`: A run of contiguous virtual memory pages carved into **fixed-sized objects**
  * Used for allocating small objects (up to 32KB in size)
  * The `spanClass` contains two parts: one identifying **sizeclass** and the least significant bit
    identifying if pointer types can be allocated within the span (it can when 0th bit is 1)
  * The size of the object is identified by the **sizeclass** of the mspan and is statically defined.
  * The number of objects in a span depends on the number of pages allocated to it. This is also statically defined
* `mcentral`: When `mcache` runs out of objects to allocate within its cached `mspan`, it requests this
  per-heap structure to give a new `mspan` to "refill" its cache. This could also lead to allocation of
  new set of pages or even asking the OS to extend the heap mapping if necessary.
* `pageCache`: A cache of upto 64 pages from which span allocation might happen.
* `heapArena`: A 64MB region in the heap. From the Go runtime perspective, expanding the heap means allocating
  a new heap arena. From the OS perspective, that would be mapping an additional 64MB virtual memory region to
  the Go process.
* `pageAlloc`: Maintains per-page information in the mapped heap.
* `sysAlloc`: Responsible for getting virtual memory pages mapped from the system. The exact syscall used
  depends on the platform. On 64-bit Linux (amd64), that would be `mmap`.

It also has dynamic data structures and they are managed by internal allocators which are quite simple to
understand and implement. These are described in the appendix [on memory management 
internal allocators](#memory-management-internal-allocators).

I mentioned earlier that allocating slice `[]T` might call the memory manager (`mallocgc`) as part of
expanding the slice. However, `mheap` also has **off-heap** slice data structures and the memory manager 
**CANNOT** call `append` because it might be a recursive call back to itself. The memory backing these 
slices are allocated from `persistentalloc` as it is **off-heap**. And `map[K]V` are not used at all
because it is much more complex than the slice type.

### `mheap`: the main heap data structure
This is the global data structure used by the allocator.
```go
var mheap_ mheap
```
All other allocations mentioned earlier `mspan`, `mcache` etc are registered here. Naturally, garbage
collector also starts from here.

### OS Memory Management Abstraction layer (sysAlloc)
Different OSes have different syscalls to allocate/map virtual memory to their address space. In 64-bit Linux
(amd64 architecture), `mmap` syscall is used. The address space managed by the runtime can be in different
states which determine whether the program can access it safely or not. They are described
[here](https://github.com/golang/go/blob/de5b418bea70aaf27de1f47e9b5813940d1e15a4/src/runtime/mem.go#L9-L48)
and I will summarise it here

| State | Description |
|-------|-------------|
| `None` | Unreserved and unmapped |
| `Reserved` | Owned by the runtime, but not accessible, doesn't count against footprint. |
| `Prepared` | Reserved, but accessing it is undefined behaviour. |
| `Ready` | Accessed safely. |

The transitions between the states are performed by various `sys*` functions defined in src/runtime/mem.go and
are implemented with different flags to `mmap`,`munmap` and `madvise` syscalls. In fact, how they are called
depends on whether we are in Cgo context or not.

### `heapArena`: unit of heap expansion (64MB) 

### `pageAlloc`: allocate pages to spans etc

### `pageCache`: optimisation over pageAlloc

### `mspan`: number of pages with equal-sized small objects
An `mspan` consists of `nelem` number of equally-sized objects. The size of the object depends on
the size class of the span. It is indicated by the `spanclass` member. This is used for small objects
up to the size of 32KB only.

### `mcentral`: per heap collection of spans
From the allocation algorithm perspective, `mcentral` supplies per-M `mcache` structures with the `mspan`

### `mcache`: per-M allocator for small objects

### `tinyalloc`: bump-the-pointer allocator for objects <16B in mcache

## Appendix
This section explains topics that are important for a deeper understanding of the allocator algorithm,
but not essential if you are just scanning the surface. This can be skipped if you'd like to

### Memory-management internal allocators
Memory management routines also require dynamic data structures. However, it is not as straightforward
like it is in usual Go programs because that would be a recursive call to the subsystem. Luckily, the
data structures are not very complex. Hence, memory-management internal allocators are simple **free-list**
and **bump-the-pointer** allocators.

#### `persistentalloc`

#### `fixalloc`

### Go runtime internal functions
These functions are seen in the runtime source code. If you are interested in the implementation details
go through these. I found some of the tricks very interesting. So I'm noting them down here.

#### `alignUp(n,a)` and `alignDown(n,a)`

#### `*gcBits` data structure in `mheap`
