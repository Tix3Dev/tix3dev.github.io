---
layout: post
title: "apoptOS-Series-2: Milestone - Memory management"
date: 2022-06-06
tags:
  -  OSDev
author: Yves Vollmeier
avatar: assets/profile-sm.png
category: apoptOS-Series
---

## Foreword
In this blog post I will talk about the concepts for memory management in apoptOS and how to use them.

## Physical memory management
Generally speaking, this part is in charge of managing actual memory in RAM, i.e. the addresses used here directly correspond to memory in RAM. In order to manage that kind of memory, physical memory, we need to know how the bootloader, [Limine](https://github.com/limine-bootloader/limine), set up the memory map. To get this information, we can use the `stivale2_struct_tag_memmap` struct provided by the [stivale2 boot protocol](https://github.com/stivale/stivale).

### Bitmap based page frame allocator
Now that we know where the usable entries / memory locations are, we can store those informations in a data structure. apoptOS makes use of a bitmap (hence the name bitmap based allocator), where each bit in the bitmap corresponds to a page / block of memory (hence the name page frame allocator) through a mapping system. The value of a bit says if it's corresponding page is free or used. Note that in apoptOS pages are 4 KiB in size.

Before we can allocate or free anything, we need to initialize it with this function:
```c
void pmm_init(struct stivale2_struct *stivale2_struct);
```
It basically does everything we've talked about so far, i.e. setting up the memory map and the bitmap.

To allocate contiguous pages, we can either use this:
```c
void *pmm_alloc(size_t page_count);
```
or preferably this:
```c
void *pmm_allocz(size_t page_count);
```
Which does the same thing as `pmm_alloc` except it also memsets the memory returned to zero (-> "clean memory" -> should be preferred).

Allocating consists of finding a free bit in the bitmap, setting it to used and returning the corresponding memory.

To free contiguous pages, the following is used:
```c
void pmm_free(void *pointer, size_t page_count);
```

Freeing consists of setting the bit corresponding to the pointer passed as an argument to free again, so that the memory can be used again.

### Slab allocator
The bitmap based page frame allocator is great for page-sized allocations, but when it comes to fine-grained allocations, there would be too much memory wastage. That's why I've implemented a slab allocator, inspired by the design proposed by [Jeff Bonwick](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf). The basic concept lays in object caching, i.e. storing preallocated objects in a customly defined cache. When allocating, no real memory has to be allocated, only the address of the preallocated object has to be returned. Same goes for freeing: No real memory is freed, only the object is marked as free again.

A cache consists of a linked list of so called slabs. If one slab happens to be fully allocated, a new slab will automatically be created if specified by the user. That's why in theory, when allocating with the slab allocator, one can't run out of memory.

A slab itself consists of a freelist or in other words a linked list of control buffers (bufctl).

There are some restrictions though, as the maximum size for a cached object is 512 bytes (so called small slabs). If you want to know more about this issue, please consult the paper by Jeff Bonwick. `malloc` solves this problem as we'll see later.

Now let's learn how to use the slab allocator:

First, we are going to use slab_cache_create to create a cache for a specific task, in this example inodes for a filesystem.
```c
slab_cache_t *slab_cache_create(const char *name, size_t slab_size, slab_flags_t flags);
```

```c
// create a cache where each slab is 64 bytes in size, the cache has a description for 
// debugging: "filesystem inodes" and flags to panic if any problem occurs

slab_cache_t *filesystem_inode_cache = slab_cache_create("filesystem inodes", 64, SLAB_PANIC);
```

Now to allocate an object of size 64 from the newly created cache, we will use the follwing function:
```c
void *slab_cache_alloc(slab_cache_t *cache, slab_flags_t flags);
```
It might look something like this:
```c
// create a cache where each slab is 64 bytes in size, the cache has a description for 
// debugging: "filesystem inodes" and flags to panic if any problem occurs

slab_cache_t *filesystem_inode_cache = slab_cache_create("filesystem inodes", 64, SLAB_PANIC);

// allocate an object from the cache, set flags to panic if any problem occurs
// and automatically grow the cache if it runs out of slabs

void *inode_ptr_1 = slab_cache_alloc(filesystem_inode_cache, SLAB_PANIC | SLAB_AUTO_GROW);
```

_NOTE: We can also manually grow or reap the cache, by using the corresponding functions:_
```c
void slab_cache_grow(slab_cache_t *cache, size_t count, slab_flags_t flags);
```
```c
void slab_cache_reap(slab_cache_t *cache, slab_flags_t flags);
```

To free our freshly allocated object, we will use the following function:
```c
void slab_cache_free(slab_cache_t *cache, void *pointer, slab_flags_t flags);
```

Our final code might look something like this - I've also added a pretty neat debugging tool, to dump the cache:
```c
void slab_cache_dump(slab_cache_t *cache, slab_flags_t flags);
```
->
```c
// create a cache where each slab is 64 bytes in size, the cache has a description for 
// debugging: "filesystem inodes" and flags to panic if any problem occurs

slab_cache_t *filesystem_inode_cache = slab_cache_create("filesystem inodes", 64, SLAB_PANIC);

// allocate an object from the cache, set flags to panic if any problem occurs
// and automatically grow the cache if it runs out of slabs

void *inode_ptr_1 = slab_cache_alloc(filesystem_inode_cache, SLAB_PANIC | SLAB_AUTO_GROW);

// let's check how our cache looks like at the moment

slab_cache_dump(filesystem_inode_cache, SLAB_PANIC);

// do something nice with inode_ptr_1 :D

// we don't need our object anymore, so let's free it

slab_cache_free(filesystem_inode_cache, inode_ptr_1, SLAB_PANIC);
```

And that's how easy it is to use the slab allocator in apoptOS!

## Virtual memory management
The basic idea of virtual memory is to have a bigger address space, ranging (in our case) from 0x0 to 0xFFFFFFFFFFFFFFFF. To get from virtual memory to physical memory, we use a mapping system. On x86_64 we are provided with paging. In apoptOS, to be specific, we make use of 4-level paging. I won't go into the details of how paging works, but instead how to use the interface.

To initialize paging and map all memory regions, we'll use:
```c
void vmm_init(struct stivale2_struct *stivale2_struct);
```

The resulting virtual memory layout looks like this:
```
First 4 GiB identical:
0x0 - 0x100000000 mapped to 0x0 - 0x100000000

First 4 GiB for higher half data:
0xFFFF800000000000 - 0xFFFF800100000000 mapped to 0x0 - 0x100000000

First 4 GiB for heap:
0xFFFF900000000000 - 0xFFFF900100000000 mapped to 0x0 - 0x100000000 

First 2 GiB for higher half code:
0xFFFFFFFF80000000 - 0x0001000000000000 mapped to 0x0 - 0x80000000 

All memory map entries:
At higher half data start to all entries in memory map 
```

To just map a page, we can use:
```c
void vmm_map_page(uint64_t *page_table, uint64_t phys_page, uint64_t virt_page, uint64_t flags);
```

And to unmap a page:
```c
void vmm_unmap_page(uint64_t *page_table, uint64_t virt_page);
```

We can also map a whole range using:
```c
void vmm_map_range(uint64_t *page_table, uint64_t start, uint64_t end, uint64_t offset, uint64_t flags);
```

And to unmap a whole range:
```c
void vmm_unmap_range(uint64_t *page_table, uint64_t start, uint64_t end);
```

Whenever we need to map something outside of `src/kernel/memory/virtual/vmm.c`, we'll need to know the address of the root page table. To get it, we have to use:
```c
uint64_t *vmm_get_root_page_table(void);
```

## Heap memory
Sometimes we don't want to allocate a whole page with `pmm_alloc` or create a slab cache and allocate from it via `slab_cache_alloc`. That might be because we don't want to waste one whole page for a small allocation, so we'd have to use the slab allocator, but then the sizes might change, so we'd have to create many caches. To fix this issue, apoptOS provides a generic interface, where we don't have to initialize anything before an allocation. The size we want to allocate can also be anything arbitrary.

TL;DR apoptOS provides a heap interface through `malloc` and `free`.

In `src/kernel/kernel.c` - `kinit_all`  the following is already called, so this function will never have to be used again:
```c
void malloc_heap_init(void);
```

Allocations and frees from now on are as easy as:
```c
void *malloc(size_t size);
```
and
```c
void free(void *pointer);
```

Addresses returned by malloc are always from the heap memory region at 0xFFFF900000000000 (see memory layout in Virtual memory). 

### Detailed implementation
Last but not least, I'd like to show how I've implemented the heap in more detail, as I've come up with a quite unique way. I haven't done any benchmarks yet, but at the moment this allocator will suffise time and efficiency wise. The slab allocator might also be changed in the future to also use large slabs, so this design won't be needed anymore. Simplicity-wise I can only recommend it!

1. `malloc` is a mix between the bitmap based page frame allocator and the slab allocator. Former of which is used for allocations above 512 bytes, latter of which is used for allocations below 512 bytes.
2. `malloc_heap_init` will create 8 caches, with sizes being power of two, starting with 4 and ending with 512. apoptOS uses 4 as the smallest possible size, as there will always be metadata, which is 2 bytes already.
3. `malloc` will check if the requested size is below or above 512 (to use the appropriate allocator).

    -> size <= 512:
      - Update size by adding the size of the metadata struct
      - Convert updated size to index that'll be used to access the array of slab caches (e.g. size=5 -> index=1 (corresponding to slab size 8))

    _NOTE: This conversion is done by:_
    ```c
    size_t get_slab_cache_index(size_t size);
    ```
    _Which is a very efficient algorithm (the conditions only), compromising looks of the code. See repo for details._
      - Allocate memory with slab allocator (argument=index)
      - Put metadata struct at beginning of allocated memory - Metadata contains slab caches index

    -> size > 512
      - Update size by adding the size of the metadata struct and page aligning it
      - Convert udpated size to page count
      - Allocate memory with bitmap based page frame allocator (argument=page count)
      - Put metadata struct at beginning of allocated memory - Metadata contains page count

4. `free` will check if the pointer ends with 0xFFF bits by bitwise ANDing

    -> not true
      - Get slab caches index from metadata at pointer passed as argument
      - Free memory with slab allocator (argument=index)

    -> true
      - Get page count from metadata at pointer passed as argument
      - Free memory with bitmap based page frame allocator (argument=page count)

And that's the whole deal with memory. Of course, everything is very basic, but for now it wouldn't make sense to implement more, instead let's start adding more features to apoptOS!

## References

- https://github.com/Tix3Dev/apoptOS/blob/9a4fde1fe18abb4fb91d89a29ae21bde7c0cbea5/src/kernel/memory/physical/pmm.c
- https://github.com/Tix3Dev/apoptOS/blob/9a4fde1fe18abb4fb91d89a29ae21bde7c0cbea5/src/kernel/memory/virtual/vmm.c
- https://github.com/Tix3Dev/apoptOS/blob/9a4fde1fe18abb4fb91d89a29ae21bde7c0cbea5/src/kernel/memory/dynamic/slab.c
- https://github.com/Tix3Dev/apoptOS/blob/9a4fde1fe18abb4fb91d89a29ae21bde7c0cbea5/src/kernel/libk/malloc/malloc.c
- https://github.com/limine-bootloader/limine
- https://github.com/stivale/stivale
- https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf
