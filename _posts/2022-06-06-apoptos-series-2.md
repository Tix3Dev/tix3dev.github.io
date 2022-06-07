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
In this blog post I will talk about the concepts used by the memory management parts in apoptOS and how to use them.

## Physical memory management
Generally speaking, this part is in charge of managing actual memory in RAM, i.e. the addresses used here directly correspond to memory in RAM. In order to manage that kind of memory, physical memory, we need to know how the bootloader, [Limine](https://github.com/limine-bootloader/limine), set up the memory map. To get this information, we can use the `stivale2_struct_tag_memmap` struct provided by the [stivale2 boot protocol](https://github.com/stivale/stivale).

### Bitmap based page frame allocator
Now that we know where the usable entries / memory locations are, we can store those informations in a data structure. apoptOS makes use of a bitmap (allocator), where each bit in the bitmap corresponds to a page / block of memory (hence the name page frame allocator) through a mapping system. The value of a bit says if it's corresponding page is free or used. Note that in apoptOS pages are 4 KiB big.

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

There are some restrictions though, as the maximum size for a cached object is 512 bytes (so called small slabs). If you want to know more about this issue, please consult the paper by Jeff Bonwick. `malloc` solves this problem as we will see later.

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
~vmm memory map~

~how to interface it~

## Heap memory
intro

~how to interface it~

## References

- https://github.com/limine-bootloader/limine
- https://github.com/stivale/stivale
- https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf
