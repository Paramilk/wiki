# Stivale CSharp Barebones

What is Stivale?
----------------

**stivale** means "boot" in Italian. It is a boot protocol designed to
overcome shortcomings of common boot protocols used by hobbyist OS
developers, such as [Multiboot](https://wiki.osdev.org/Multiboot).

There are 2 revisions of the stivale boot protocol, namely: **stivale**,
and **stivale2**. Stivale2 makes use of tags for bootloader writers' and
kernel writers' convenience, and to make future expandability and
revisioning easier. We will use second version.

[Stivale GitHub repository](https://github.com/stivale/stivale)

What is Limine?
---------------

[Limine](https://wiki.osdev.org/Limine) is a modern, advanced x86 bootloader for
[BIOS](https://wiki.osdev.org/BIOS) and [UEFI](https://wiki.osdev.org/UEFI), with support for
cutting edge features such as 5-level paging, 64-bit [Long
Mode](https://wiki.osdev.org/Long_Mode), and direct higher half loading thanks to
the [stivale](https://github.com/stivale/stivale) boot protocol.

[Limine GitHub repository](https://github.com/limine-bootloader/limine)

Getting started
---------------

It is recommended to check out this project
[repository](https://github.com/ilobilo/stivale2-cs-barebones) until
getting started

### Install required packages

To compile and run the kernel you will need following programs
installed:

-   [Dotnet](https://docs.microsoft.com/en-us/dotnet/core/install/linux)
    (C\# compiler) should be installed in /usr/share/dotnet/
-   Tysila2 (Debian package included in repository)
-   [Mono](https://www.mono-project.com) (To run tysila2)
-   Make (To run everything)
-   Nasm (Assembly compiler)
-   ld.lld (Linker)
-   Xorriso (To build the iso)
-   Qemu-system-x86 (To run the OS)

If you are using debian based system you can install most of them with
these commands:

`sudo apt install nasm ld.lld xorriso qemu-system-x86 mono-runtime`\
`// First download tysila2.deb from project's repository`\
`sudo dpkg -i tysila2.deb`

### Actual code

First of all you need to create following files and directories:

`├── limine.cfg`\
`├── Makefile`\
`└── source`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`├── Console.cs`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`|── kernel.asm`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`├── kernel.cs`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`├── linker.ld`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`├── Makefile`\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`└── stivale2.cs`

#### Limine configuration file

This is [Limine](https://wiki.osdev.org/Limine) configuration file which tells
bootloader what to do:


limine.cfg
```ini
# Kernel entry name in bootloader menu
:Stivale2
# Boot protocol to use
PROTOCOL=stivale2
# Kernel elf file path on iso
KERNEL_PATH=boot:///kernel.elf
# Disable KASLR
KASLR=no
```

#### Makefile

These are simple makefile that will be excecuted when you run make
command:

Note: Make sure to use tabs instead of 8 spaces

Makefile:

```makefile
KERNELDIR := $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))

all: limine
    mkdir -p $(KERNELDIR)/iso_root
    $(MAKE) -s -C $(KERNELDIR)/source

limine:
    git clone https://github.com/limine-bootloader/limine.git --single-branch --branch=latest-binary --depth=1
    $(MAKE) -C $(KERNELDIR)/limine

clean:
    $(MAKE) -s -C $(KERNELDIR)/source clean

distclean:
    $(MAKE) -s -C $(KERNELDIR)/source clean
    rm -rf $(KERNELDIR)/limine
    rm -rf $(KERNELDIR)/iso_root
```

source/Makefile:

```makefile
SOURCEDIR := $(shell dirname $(realpath $(firstword $(MAKEFILE_LIST))))

KERNEL := $(SOURCEDIR)/kernel.elf
KERNELEXE := $(KERNEL:.elf=.exe)
KERNELO := $(KERNEL:.elf=.o)
ISO = $(SOURCEDIR)/../image.iso

LIMINE = $(SOURCEDIR)/../limine/limine-install

QEMU = qemu-system-x86_64
QEMUFLAGS = -enable-kvm -M q35 -cpu max -smp 2 -m 512M -boot d -rtc base=localtime -serial stdio

XORRISO = xorriso
XORRISOFLAGS = -as mkisofs -b limine-cd.bin \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        --efi-boot limine-eltorito-efi.bin -efi-boot-part \
        --efi-boot-image --protective-msdos-label

LD = ld.lld
AS = nasm
AOT = tysila2
DOTNET = /usr/share/dotnet/dotnet
CSC = /usr/share/dotnet/sdk/*/Roslyn/bincore/csc.dll

ASMFLAGS = -f elf64
AOTFLAGS = --arch x86_64-elf64-tysos -fno-rtti -fno-exceptions
CSCFLAGS = -unsafe -target:exe -platform:x86 -nostdlib /r:/usr/share/tysila2/mscorlib.dll

LDFLAGS = -T $(SOURCEDIR)/linker.ld -m elf_x86_64 -z max-page-size=0x1000

CSFILES = $(shell find $(SOURCEDIR)/ -type f -name '*.cs')
ASMFILES = $(shell find $(SOURCEDIR)/ -type f -name '*.asm')
OBJ = $(ASMFILES:.asm=_asm.o)

.PHONY: all
all: $(KERNEL)
    $(MAKE) iso
    $(MAKE) clean run

$(KERNEL): $(OBJ)
    $(DOTNET) $(CSC) $(CSCFLAGS) $(CSFILES) -out:$(KERNELEXE)
    $(AOT) $(AOTFLAGS) $(KERNELEXE) -o $(KERNELO)
    $(LD) $(LDFLAGS) $(INTERNALLDFLAGS) $(OBJ) $(KERNELO) -o $@

%_asm.o: %.asm
    $(AS) $(ASMFLAGS) $^ -o $@

iso:
    cp $(KERNEL) $(SOURCEDIR)/../limine.cfg $(SOURCEDIR)/../limine/limine.sys \
        $(SOURCEDIR)/../limine/limine-cd.bin $(SOURCEDIR)/../limine/limine-eltorito-efi.bin $(SOURCEDIR)/../iso_root/
    $(XORRISO) $(XORRISOFLAGS) $(SOURCEDIR)/../iso_root -o $(ISO)
    $(LIMINE) $(ISO)

clean:
    rm -rf $(KERNEL) $(OBJ) $(KERNELEXE) $(KERNELO) $(SOURCEDIR)/../iso_root/*

run:
    $(QEMU) $(QEMUFLAGS) -cdrom $(ISO)
```

#### Linker script

Script that ld.lld will use to link relocatable elf objects

source/linker.ld

```
OUTPUT_FORMAT(elf64-x86-64)
OUTPUT_ARCH(i386:x86-64)

ENTRY(_start)

PHDRS
{
    null    PT_NULL    FLAGS(0) ;
    text    PT_LOAD    FLAGS((1 << 0) | (1 << 2)) ;
    rodata  PT_LOAD    FLAGS((1 << 2)) ;
    data    PT_LOAD    FLAGS((1 << 1) | (1 << 2)) ;
    dynamic PT_DYNAMIC FLAGS((1 << 1) | (1 << 2)) ;
}

SECTIONS
{
    . = 0xffffffff80200000;

    .text : {
        *(.text*)
    } :text

    . += 0x1000;

    .stivale2hdr : {
        KEEP(*(.stivale2hdr))
    } :rodata

    .rodata : {
        *(.rodata*)
    } :rodata

    . += 0x1000;

    .data : {
        *(.data*)
    } :data

    .dynamic : {
        *(.dynamic)
    } :data :dynamic

    .bss : {
        *(COMMON)
        *(.bss*)
    } :data
}
```

#### Kernel

These files contain code that will be executed when kernel runs:

source/kernel.asm

```as
global sthrow

; C# entry point
extern _ZN6kernel6Kernel7ProgramM_0_8RealMain_Rv_P1PV26stivale2#2Bstivale2_struct

section .data

stivale2_smp_tag:
    dq 0x1ab015085f3273df
    dq 0
    dq 0

; Framebuffer
stivale2_framebuffer_tag:
    dq 0x3ecc1bc43d0f7971
    dq stivale2_smp_tag
    dw 0
    dw 0
    dw 32

; Text mode
stivale2_any_video_tag:
    dq 0xc75c9fa92a44c4db
    dq stivale2_smp_tag
    dq 1

; Kernel stack
section .bss
align 16
stack_bottom:
resb 8192
stack_top:

; Stivale2 header
section .stivale2hdr
align 4
stivale_hdr:
    dq kmain
    dq stack_top
    dq (1 << 1)
    ; Replace this with "stivale2_framebuffer_tag" to enable framebuffer
    dq stivale2_any_video_tag

section .text
kmain:
    push rdi
    call _ZN6kernel6Kernel7ProgramM_0_8RealMain_Rv_P1PV26stivale2#2Bstivale2_struct

sthrow:
    hlt
    jmp sthrow
```

source/kernel.cs

```csharp
namespace Kernel
{
    // Class is unsafe so we can use pointers
    public unsafe class Program
    {
        // Fake kernel entry point
        // Do not remove
        public static void Main()
        {
            // These lines of code are required
            // Without them compiler will remove RealMain from kernel
            stivale2.stivale2_struct* test = null;
            RealMain(test);
        }

        public static stivale2.stivale2_struct_tag_smp *smp_tag;

        // Real entry
        public static void RealMain(stivale2.stivale2_struct* stiv)
        {
            // If Stivale2 struct was not found halt
            if (stiv == null) while (true);

            // Example on how to get Stivale2 structure tags
            smp_tag = (stivale2.stivale2_struct_tag_smp*)stivale2.get_tag(stiv, stivale2.STIVALE2_STRUCT_TAG_SMP_ID);

            // Print text
            Console.WriteLine("Hello, World!");

            // Halt
            while (true);
        }
    }
}
```

#### Console

This is an example code that provides console write and writeline
functions: source/Console.cs

```csharp
namespace Kernel
{
    // Console colours
    public enum ConsoleColour
    {
        Black,
        Blue,
        Green,
        Cyan,
        Red,
        Purple,
        Brown,
        Grey,
        DarkGrey,
        LightBlue,
        LightGreen,
        LightCyan,
        LightRed,
        LightPurple,
        Yellow,
        White,
    };
    public unsafe static class Console
    {
        // VGA text mode address
        public static ushort* vga = (ushort*)0xB8000;
        // Cursor positions
        public static int x = 0, y = 0, lastx = 0;
        // Text colours
        public static ConsoleColour ForegroundColour = ConsoleColour.White;
        public static ConsoleColour BackgroundColour = ConsoleColour.Black;

        public static void Write(string s)
        {
            foreach(char c in s)
            {
                PutChar(c, BackgroundColour, ForegroundColour);
            }
        }

        public static void WriteLine(string s)
        {
            foreach(char c in s)
            {
                PutChar(c, BackgroundColour, ForegroundColour);
            }
            PutChar('\n', BackgroundColour, ForegroundColour);
        }

        static void PutChar(char c, ConsoleColour bgcolour, ConsoleColour fgcolour)
        {
            switch (c)
            {
                // Newline
                case '\n':
                    lastx = x;
                    x = 0;
                    y++;
                    break;
                // Backspace
                case '\b':
                    if (x > 0)
                    {
                        x--;
                        vga[y * 80 + x] = (ushort)(((byte)bgcolour << 12) | ((byte)fgcolour << 8) | ' ');
                    }
                    else
                    {
                        x = lastx;
                        y--;
                    }
                    break;
                // Everything else
                default:
                    vga[y * 80 + x] = (ushort)(((byte)bgcolour << 12) | ((byte)fgcolour << 8) | c);
                    if (x < 80) x++;
                    else
                    {
                        lastx = x;
                        x = 0;
                        y++;
                    }
                    break;
            }
        }
    }
}
```

#### Stivale2.cs

[download](https://github.com/ilobilo/stivale2-cs-barebones/blob/main/source/stivale2.cs)
it and place under source/

### Building and running the kernel

To compile and run the kernel go to main directory (Where limine.cfg,
Makefile and source/ are) and run:

`make -j$(nproc --all)`

See Also
--------

-   [Stivale2
    Specification](https://github.com/stivale/stivale/blob/master/STIVALE2.md)
-   [Limine bootloader](https://github.com/limine-bootloader/limine)
-   [Project github
    repository](https://github.com/ilobilo/stivale2-cs-barebones/)
