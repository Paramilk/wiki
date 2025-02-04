# Stivale Barebones

**stivale** means "boot" in Italian. It is a boot protocol designed to
overcome shortcomings of common boot protocols used by hobbyist OS
developers, such as [Multiboot](https://wiki.osdev.org/Multiboot).

There are actually 2 revisions of the stivale boot protocol, namely:
**stivale**, and **stivale2**. The earlier stivale revision (simply
"stivale", but for clarity we are going to call it "stivale1" from now
on), is a very simple "KISS" boot protocol only supporting what was
deemed necessary at the time, but versioning and expandability issues
made creating stivale2 a necessity. stivale2 makes use of tags for
bootloader writers' and kernel writers' convenience, and to make future
expandability and revisioning easier.

This article only covers stivale2, as stivale1 is deprecated and should
be avoided for new kernels.

stivale is firmware and architecture agnostic, though the only
bootloader fully supporting the stivale protocols as of writing this
article (2021/01/09) is [Limine](https://wiki.osdev.org/Limine), which is an
x86/x86\_64 [BIOS](https://wiki.osdev.org/BIOS) and [UEFI](https://wiki.osdev.org/UEFI)
bootloader, with [TomatBoot](https://github.com/TomatOrg/TomatBoot) (now
archived) and [Sabaton](https://github.com/FlorenceOS/Sabaton) offering
limited stivale compliance for x86\_64 [UEFI](https://wiki.osdev.org/UEFI)
exclusively, and aarch64, respectively.

This article will demonstrate how to write a small 64-bit higher half
stivale2 kernel in (GNU) C, and boot it using the
[Limine](https://wiki.osdev.org/Limine) bootloader.

It is also recommended to check out
[this](https://github.com/limine-bootloader/limine-barebones) project as
it provides example buildable code to go along with this guide.

## Overview

For this example, we will create these 2 files and place them in the
same directory:

-   kernel.c
-   linker.ld

As one may notice, there is no "entry point" assembly stub, as one is
not necessary with either stivale1 or 2, when using a language which can
make use of a standard SysV x86 [calling
convention](https://wiki.osdev.org/Calling_Conventions).

Furthermore, we will download the header file **stivale2.h** which
defines structures that we will use to interact with the bootloader from
[here](https://raw.githubusercontent.com/stivale/stivale/master/stivale2.h),
and place it in the same directory as the other files.

Obviously, this is just a bare bones example, and one should always
refer to the [stivale2
specification](https://github.com/stivale/stivale/blob/master/STIVALE2.md)
for more details and information.

### kernel.c

This is the kernel "main".

```c
#include <stdint.h>
#include <stddef.h>
#include <stivale2.h>

// We need to tell the stivale bootloader where we want our stack to be.
// We are going to allocate our stack as an array in .bss.
static uint8_t stack[8192];

// stivale2 uses a linked list of tags for both communicating TO the
// bootloader, or receiving info FROM it. More information about these tags
// is found in the stivale2 specification.

// stivale2 offers a runtime terminal service which can be ditched at any
// time, but it provides an easy way to print out to graphical terminal,
// especially during early boot.
// Read the notes about the requirements for using this feature below this
// code block.
static struct stivale2_header_tag_terminal terminal_hdr_tag = {
    // All tags need to begin with an identifier and a pointer to the next tag.
    .tag = {
        // Identification constant defined in stivale2.h and the specification.
        .identifier = STIVALE2_HEADER_TAG_TERMINAL_ID,
        // If next is 0, it marks the end of the linked list of header tags.
        .next = 0
    },
    // The terminal header tag possesses a flags field, leave it as 0 for now
    // as it is unused.
    .flags = 0
};

// We are now going to define a framebuffer header tag.
// This tag tells the bootloader that we want a graphical framebuffer instead
// of a CGA-compatible text mode. Omitting this tag will make the bootloader
// default to text mode, if available.
static struct stivale2_header_tag_framebuffer framebuffer_hdr_tag = {
    // Same as above.
    .tag = {
        .identifier = STIVALE2_HEADER_TAG_FRAMEBUFFER_ID,
        // Instead of 0, we now point to the previous header tag. The order in
        // which header tags are linked does not matter.
        .next = (uint64_t)&terminal_hdr_tag
    },
    // We set all the framebuffer specifics to 0 as we want the bootloader
    // to pick the best it can.
    .framebuffer_width  = 0,
    .framebuffer_height = 0,
    .framebuffer_bpp    = 0
};

// The stivale2 specification says we need to define a "header structure".
// This structure needs to reside in the .stivale2hdr ELF section in order
// for the bootloader to find it. We use this __attribute__ directive to
// tell the compiler to put the following structure in said section.
__attribute__((section(".stivale2hdr"), used))
static struct stivale2_header stivale_hdr = {
    // The entry_point member is used to specify an alternative entry
    // point that the bootloader should jump to instead of the executable's
    // ELF entry point. We do not care about that so we leave it zeroed.
    .entry_point = 0,
    // Let's tell the bootloader where our stack is.
    // We need to add the sizeof(stack) since in x86(_64) the stack grows
    // downwards.
    .stack = (uintptr_t)stack + sizeof(stack),
    // Bit 1, if set, causes the bootloader to return to us pointers in the
    // higher half, which we likely want since this is a higher half kernel.
    // Bit 2, if set, tells the bootloader to enable protected memory ranges,
    // that is, to respect the ELF PHDR mandated permissions for the executable's
    // segments.
    // Bit 3, if set, enables fully virtual kernel mappings, which we want as
    // they allow the bootloader to pick whichever *physical* memory address is
    // available to load the kernel, rather than relying on us telling it where
    // to load it.
    // Bit 4 disables a deprecated feature and should always be set.
    .flags = (1 << 1) | (1 << 2) | (1 << 3) | (1 << 4),
    // This header structure is the root of the linked list of header tags and
    // points to the first one in the linked list.
    .tags = (uintptr_t)&framebuffer_hdr_tag
};

// We will now write a helper function which will allow us to scan for tags
// that we want FROM the bootloader (structure tags).
void *stivale2_get_tag(struct stivale2_struct *stivale2_struct, uint64_t id) {
    struct stivale2_tag *current_tag = (void *)stivale2_struct->tags;
    for (;;) {
        // If the tag pointer is NULL (end of linked list), we did not find
        // the tag. Return NULL to signal this.
        if (current_tag == NULL) {
            return NULL;
        }

        // Check whether the identifier matches. If it does, return a pointer
        // to the matching tag.
        if (current_tag->identifier == id) {
            return current_tag;
        }

        // Get a pointer to the next tag in the linked list and repeat.
        current_tag = (void *)current_tag->next;
    }
}

// The following will be our kernel's entry point.
void _start(struct stivale2_struct *stivale2_struct) {
    // Let's get the terminal structure tag from the bootloader.
    struct stivale2_struct_tag_terminal *term_str_tag;
    term_str_tag = stivale2_get_tag(stivale2_struct, STIVALE2_STRUCT_TAG_TERMINAL_ID);

    // Check if the tag was actually found.
    if (term_str_tag == NULL) {
        // It wasn't found, just hang...
        for (;;) {
            asm ("hlt");
        }
    }

    // Let's get the address of the terminal write function.
    void *term_write_ptr = (void *)term_str_tag->term_write;

    // Now, let's assign this pointer to a function pointer which
    // matches the prototype described in the stivale2 specification for
    // the stivale2_term_write function.
    void (*term_write)(const char *string, size_t length) = term_write_ptr;

    // We should now be able to call the above function pointer to print out
    // a simple "Hello World" to screen.
    term_write("Hello World", 11);

    // We're done, just hang...
    for (;;) {
        asm ("hlt");
    }
}
```

**Note:** Using the stivale2 terminal requires that the kernel maintains
some state as described in the specification
(https://github.com/stivale/stivale/blob/master/STIVALE2.md\#x86\_64-1).

### linker.ld

This is going to be our linker script describing where our sections will
end up in memory.

```c
/* Tell the linker that we want an x86_64 ELF64 output file */
OUTPUT_FORMAT(elf64-x86-64)
OUTPUT_ARCH(i386:x86-64)

/* We want the symbol _start to be our entry point */
ENTRY(_start)

/* Define the program headers we want so the bootloader gives us the right */
/* MMU permissions */
PHDRS
{
    null    PT_NULL    FLAGS(0) ;                   /* Null segment */
    text    PT_LOAD    FLAGS((1 << 0) | (1 << 2)) ; /* Execute + Read */
    rodata  PT_LOAD    FLAGS((1 << 2)) ;            /* Read only */
    data    PT_LOAD    FLAGS((1 << 1) | (1 << 2)) ; /* Write + Read */
}

SECTIONS
{
    /* We wanna be placed in the topmost 2GiB of the address space, for optimisations */
    /* and because that is what the stivale2 spec mandates. */
    /* Any address in this region will do, but often 0xffffffff80000000 is chosen as */
    /* that is the beginning of the region. */
    . = 0xffffffff80000000;

    .text : {
        *(.text*)
    } :text

    /* Move to the next memory page for .rodata */
    . += CONSTANT(MAXPAGESIZE);

    /* We place the .stivale2hdr section containing the header in its own section, */
    /* and we use the KEEP directive on it to make sure it doesn't get discarded. */
    .stivale2hdr : {
        KEEP(*(.stivale2hdr))
    } :rodata

    .rodata : {
        *(.rodata*)
    } :rodata

    /* Move to the next memory page for .data */
    . += CONSTANT(MAXPAGESIZE);

    .data : {
        *(.data*)
    } :data

    .bss : {
        *(COMMON)
        *(.bss*)
    } :data
}
```

Building the kernel and creating an image
-----------------------------------------

### Makefile

In order to build our kernel, we are going to use a Makefile.

```makefile
# This is the name that our final kernel executable will have.
# Change as needed.
KERNEL := myos.elf

# It is highly recommended to use a custom built cross toolchain to build a kernel.
# We are only using "cc" as a placeholder here. It may work by using
# the host system's toolchain, but this is not guaranteed.
CC ?= cc

# Likewise, "ld" here is just a placeholder and your mileage may vary if using the
# host's "ld".
LD ?= ld

# User controllable CFLAGS.
CFLAGS ?= -Wall -Wextra -O2 -pipe

# User controllable linker flags. We set none by default.
LDFLAGS ?=

# Internal C flags that should not be changed by the user.
INTERNALCFLAGS :=            \
    -I.                  \
    -std=gnu11           \
    -ffreestanding       \
    -fno-stack-protector \
    -fno-pic             \
    -mno-80387           \
    -mno-mmx             \
    -mno-3dnow           \
    -mno-sse             \
    -mno-sse2            \
    -mno-red-zone        \
        -mcmodel=kernel      \
        -MMD

# Internal linker flags that should not be changed by the user.
INTERNALLDFLAGS :=             \
    -Tlinker.ld            \
    -nostdlib              \
    -zmax-page-size=0x1000 \
    -static

# Use find to glob all *.c files in the directory and extract the object names.
CFILES := $(shell find ./ -type f -name '*.c')
OBJ := $(CFILES:.c=.o)
HEADER_DEPS := $(CFILES:.c=.d)

# Default target.
.PHONY: all
all: $(KERNEL)

# Link rules for the final kernel executable.
$(KERNEL): $(OBJ)
    $(LD) $(OBJ) $(LDFLAGS) $(INTERNALLDFLAGS) -o $@

# Compilation rules for *.c files.
-include $(HEADER_DEPS)
%.o: %.c
    $(CC) $(CFLAGS) $(INTERNALCFLAGS) -c $< -o $@

# Remove object files and the final executable.
.PHONY: clean
clean:
    rm -rf $(KERNEL) $(OBJ) $(HEADER_DEPS)
```

### limine.cfg

This file is parsed by Limine and it describes boot entries and other
bootloader configuration variables. Further information
[here](https://github.com/limine-bootloader/limine/blob/trunk/CONFIG.md).

```ini
# Timeout in seconds that Limine will use before automatically booting.
TIMEOUT=5

# The entry name that will be displayed in the boot menu
:myOS

# Change the protocol line depending on the used protocol.
PROTOCOL=stivale2

# Path to the kernel to boot. boot:/// represents the partition on which limine.cfg is located.
KERNEL_PATH=boot:///myos.elf
```

## Compiling the kernel

We can now build our example kernel by running **make**. This command,
if successful, should generate a file called **myos.elf** (or the chosen
kernel name). This is our stivale2-compliant kernel executable.

## Creating the image

We can now create either an ISO or a hard disk/USB drive image with our
kernel on it. [Limine](https://wiki.osdev.org/Limine) can boot on both
[BIOS](https://wiki.osdev.org/BIOS) and [UEFI](https://wiki.osdev.org/UEFI) if the image is set
up to do so, which is what we are going to do.

### Creating an ISO

In this example we are going to create a CD-ROM ISO capable of booting
on both [UEFI](https://wiki.osdev.org/UEFI) and legacy [BIOS](https://wiki.osdev.org/BIOS)
systems.

For this to work, we will need the **xorriso** utility.

These are shell commands. They can also be compiled into a script or
Makefile.

```bash
# Download the latest Limine binary release.
git clone https://github.com/limine-bootloader/limine.git --branch=v2.0-branch-binary --depth=1

# Build limine-install.
make -C limine

# Create a directory which will be our ISO root.
mkdir -p iso_root

# Copy the relevant files over.
cp -v myos.elf limine.cfg limine/limine.sys \
      limine/limine-cd.bin limine/limine-eltorito-efi.bin iso_root/

# Create the bootable ISO.
xorriso -as mkisofs -b limine-cd.bin \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        --efi-boot limine-eltorito-efi.bin \
        -efi-boot-part --efi-boot-image --protective-msdos-label \
        iso_root -o image.iso

# Install Limine stage 1 and 2 for legacy BIOS boot.
./limine/limine-install image.iso
```

### Creating a hard disk/USB drive image

In this example we'll create a [GPT](https://wiki.osdev.org/GPT) partition table
using **parted**, containing a single FAT partition, also known as the
ESP in EFI terminology, which will store our kernel, configs, and
bootloader.

This example is more involved and is made up of more steps than creating
an ISO image.

These are shell commands. They can also be compiled into a script or
Makefile.

```bash
# Create an empty zeroed out 64MiB image file.
dd if=/dev/zero bs=1M count=0 seek=64 of=image.hdd

# Create a GPT partition table.
parted -s image.hdd mklabel gpt

# Create an ESP partition that spans the whole disk.
parted -s image.hdd mkpart ESP fat32 2048s 100%
parted -s image.hdd set 1 esp on

# Download the latest Limine binary release.
git clone https://github.com/limine-bootloader/limine.git --branch=v2.0-branch-binary --depth=1

# Build limine-install.
make -C limine

# Install the Limine BIOS stages onto the image.
./limine/limine-install image.hdd

# Mount the loopback device.
USED_LOOPBACK=$(sudo losetup -Pf --show image.hdd)

# Format the ESP partition as FAT32.
sudo mkfs.fat -F 32 ${USED_LOOPBACK}p1

# Mount the partition itself.
mkdir -p img_mount
sudo mount ${USED_LOOPBACK}p1 img_mount

# Copy the relevant files over.
sudo mkdir -p img_mount/EFI/BOOT
sudo cp -v myos.elf limine.cfg limine/limine.sys img_mount/
sudo cp -v limine/BOOTX64.EFI img_mount/EFI/BOOT/

# Sync system cache and unmount partition and loopback device.
sync
sudo umount img_mount
sudo losetup -d ${USED_LOOPBACK}
```

Conclusions
-----------

If everything above has been completed successfully, you should now have
a bootable ISO or hard drive/USB image containing your 64-bit higher
half stivale2 kernel and Limine to boot it. Once the kernel is
successfully booted, you should see "Hello World" printed on screen.

See Also
--------

### External Links

-   [stivale2
    Specification](https://github.com/stivale/stivale/blob/master/STIVALE2.md)
-   [The sabaton aarch64 stivale2
    bootloader](https://github.com/FlorenceOS/Sabaton)
-   [The TomatBoot x86\_64 UEFI stivale1 and 2
    bootloader](https://github.com/TomatOrg/TomatBoot) (now archived)
