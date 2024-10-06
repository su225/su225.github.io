---
title: "Paper notes: Roaring bitmap"
date: 2024-09-20T19:00:17+05:30
draft: true
tags: [data-structures,paper-notes]
---

Link to the papers
1. [Better bitmap performance with Roaring bitmaps](https://arxiv.org/pdf/1402.6407)
1. [Consistently faster and smaller compressed bitmaps with Roaring](https://arxiv.org/pdf/1603.06549)
1. [Roaring Bitmaps: Implementation of an Optimized Software Library](https://arxiv.org/pdf/1709.07821v4)

{{< toc >}}

## Summary
1. Naive bitmaps consume too much memory when bitmaps are sparse and the range is large
1. Use square-root decomposition like technique on sets with cardinality of at most 2^32^ elements and the 32-bit number is divided into two 16-bit parts
    1. First 16-bit represents the chunk index where each chunk represents the range [0,2^16^), [2^16^, 2^17^) and so on
    1. Second 16-bit is the actual value
1. Multiple chunk representations
    1. **Sparse**: Sorted array of 16-bit integers representing set bit positions
    1. **Dense**: The usual bitset for the whole range (fits in 2^16^/8 = 2^13^ = 8KB)
    1. **Run-length encoding**: Described in paper-2
1. When to use what representation. Size cutoff = 4096
    1. Why? Dense bitset represents the whole range in 8KB. Each entry in the sparse representation consumes 2 bytes (16-bit). So 8KB/2 = 4096 is the maximum number of entries
    1. When number of elements is <= 4096, use sparse or use good old dense.
1. Set operations - union, intersection and difference need to consider different representations

## Representation

## Set operations
Set operations between two bitsets happen at the chunk level - i.e chunks with the same chunk index. This also means that the chunks can be processed in parallel to speed up the operations

Insertion and deletion require computing chunk indexes to jump to the chunk. Given the `u32` item, here's how to extract the `chunk_index` and the `container_element` position
```rs
#[inline]
fn chunk_index(item: u32) -> u16 {
    ((item & 0xffff_0000) >> 16) as u16
}

#[inline]
fn container_element(item: u32) -> u16 {
    (item & 0x0000_ffff) as u16
}
```

### Insertion into the set
Algorithm
1. Compute `chunk_index` and `container_element`
1. If the chunk at the `chunk_index` does not exist then create a sparse one and insert it
1. Otherwise, insert into the selected chunk
    1. If sparse, then insert the `container_element` into the sorted set. If the resulting length is more than 4096 then it has to be converted into the dense representation
    1. If dense, set the bit position and we are done

### Deletion into the set

### Union
1. **Bitmap v Bitmap**: Good old bitwise OR of all the elements. The resulting container would also be dense because if a bit is set in one, it will be in another.

1. **Bitset v Array**: The simplest algorithm is to start with a copy of the bitset, then iterate through the sorted array and turn on the respective bits in the result. The resulting container type will always be a dense bitset.

1. **Array v Array**: Merge algorithm from the merge sort suffices. But the size of the union could exceed 4096. So, the container might have to be converted to dense.

### Intersection

### Symmetric difference

