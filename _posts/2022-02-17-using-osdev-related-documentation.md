---
layout: post
title: "Using OSDev-related documentation"
date: 2021-10-21
tags:
  -  OSDev
  -  Programming
author: Yves Vollmeier
avatar: assets/profile-sm.png
category: Programming
---

## Foreword:

It can be a quite daunting task for newcomers to the OSDev world to read documentation like the Intel System Programming Guide or UEFI specifications.
Even once the desired part is found in the sheer endless documentation, it's still hard to make use of those technical information's.

As I am slowly getting better at extracting such information's, I wanted to share a few tips and tricks using an example.

## Example: Implementing RSDP structure for ACPI

In case you don't know yet, in order to parse ACPI tables like MADT etc., it's necessary to find the "Root System Description Pointer" (short: RSDP). After that, we need to extract information from it.

So what I did, as hopefully every programmer would do, was to search for this structure (in my case in the UEFI ACPI specification). And fortunately, also found it:

![RSDP Structure from UEFI ACPI specification](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/UEFI_Spec_RSDP_Structure.png)

First step done!

Next step is to read all information's contained in this table. It's really worth reading it carefully and trying to understand everything (although 100% isn't necessary, as it's already very helpful just having heard the terms for the future).
But as always, if you don't understand a key part, you can just search for it on the internet. E.g., I had to search for OEM and thanks to that, I now know that it stands for "Original equipment manufacturer" and a common example are computer manufacturers that integrate OEM parts such as processors or software into their own product.

That's cool and all, but we still need to "convert" this table into code, which is our last step.

I started off by writing some pseudocode, which eventually will turn into C code:

```c
typedef struct
{
    // ACPI version 1.0
    var signature;
    var checksum;
    var oemid;
    var revision;
    var rsdt_address;

    // ACPI version 2.0+
    var length;
    var xsdt_address;
    var extended_checksum;
    var reserved;

} rsdp_structure_t;
```

(The fact that this is a struct just comes from the ACPI context.)

As you can see, for now I only defined a struct with non-existing data types. Our main goal now is to change them into proper ones.

Let's start changing the easy ones, whose types are described in the "Description" part:

![Easy_Items](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/Easy_Items.png)

->

```c
typedef struct
{
    // ACPI version 1.0
    var signature;
    var checksum;
    var oemid;
    var revision;
    uint32_t rsdt_address;  // <-

    // ACPI version 2.0+
    var length;
    uint64_t xsdt_address;  // <-
    var extended_checksum;
    var reserved;

} rsdp_structure_t;
```

As we can see and will see, it's very important to know how big a data type is.
This should help fresh things up:
```c
// Some important C data types for the x86_64 architecture

char var1;	// 1 byte
short var2;	// 2 bytes
int var3;	// 4 bytes
long var4;	// 8 bytes

// Generally what would be preferred in OSDev

uint8_t var5;	// 8 bits   = 1 byte
uint16_t var6;	// 16 bits  = 2 bytes
uint32_t var7;	// 32 bits  = 4 bytes
uint64_t var8;	// 64 bits  = 8 bytes
```

Next up we can tell by the naming that the following ones are a string and we also get the byte length that they take up, which is just the `[n]`, as one char is one byte:

![String_Items](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/String_Items.png)

->

```c
typedef struct
{
    // ACPI version 1.0
    char signature[8];	// <-
    var checksum;
    char oemid[6];	// <-
    var revision;
    uint32_t rsdt_address;

    // ACPI version 2.0+
    var length;
    uint64_t xsdt_address;
    var extended_checksum;
    var reserved;

} rsdp_structure_t;
```

If we have a look at the byte length column, we can see that the following ones have the same length (namely 1 byte = 8 bits):

![One_Byte_Length_Items](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/One_Byte_Length_Items.png)

->

```c
typedef struct
{
    // ACPI version 1.0
    char signature[8];
    uint8_t checksum;	// <-
    char oemid[6];
    uint8_t revision;	// <-
    uint32_t rsdt_address;

    // ACPI version 2.0+
    var length;
    uint64_t xsdt_address;
    uint8_t extended_checksum;	// <-
    var reserved;

} rsdp_structure_t;
```

The table in the documentation says, that the field "Length", has a byte length of 4 bytes = 32 bits

![Length_Item](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/Length_Item.png)

->

```c
typedef struct
{
    // ACPI version 1.0
    char signature[8];
    uint8_t checksum;
    char oemid[6];
    uint8_t revision;
    uint32_t rsdt_address;

    // ACPI version 2.0+
    uint32_t length;	// <-
    uint64_t xsdt_address;
    uint8_t extended_checksum;
    var reserved;

} rsdp_structure_t;
```

Finally, we've got the field "Reserved", which has a byte length of 3. Oh no, what now? We don't have a data type with that size... Actually, that's an easy fix, let's just use 3 uint8_t -> 3 * 1 byte = 3 bytes.

![Reserved](https://raw.githubusercontent.com/Tix3Dev/tix3dev.github.io/main/_posts/2022-02-17-using-osdev-related-documentation-pictures/Reserved_Item.png)

->

```c
typedef struct
{
    // ACPI version 1.0
    char signature[8];
    uint8_t checksum;
    char oemid[6];
    uint8_t revision;
    uint32_t rsdt_address;

    // ACPI version 2.0+
    uint32_t length;
    uint64_t xsdt_address;
    uint8_t extended_checksum;
    uint8_t reserved[3];

} rsdp_structure_t;
```

And that's it, our final struct, ready to use for ACPI table parsing!

## References

- https://uefi.org/sites/default/files/resources/ACPI_6_3_final_Jan30.pdf
