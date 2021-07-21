# The Beginning: Entry and Paging

## xv6's Memory Layout

The whole point of virtualizing memory is to give users the illusion that they
can roam freely across a limitless field of memory without worrying their pretty
little heads about such boring details as how much physical memory their machine
actually has, or where kernel code is stored, or the fact that their seemingly-
continuous heap space is actually shattered into tons of tiny pages spread out
in possibly random parts of physical memory. As long as user code is well-
behaved, that illusion should hold up; if they do a no-no we'll just smack them
with a segmentation fault.

One downside is that the kernel also has to use virtual memory, so we're faced
with the potentially-complicated challenge of setting things up in physical
memory without knowing where anything is actually located in physical memory! So
xv6 does something that a lot of OSes do: it sets itself up as a higher-half
kernel. That means that in the virtual address space (from 0 to 4 GB), the
kernel will reside in the upper half starting at 2 GB, i.e. address 0x8000_0000
and up; user code will start at 0 and end at 2 GB. Because of this, `KERNBASE`
is defined in [memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h) as 0x8000_0000.

Then it sets up paging so that all of physical memory is identity-mapped to
virtual memory starting at 0x8000_0000. This makes it really convenient for the
kernel to figure out the physical address of a virtual address it's using; just
subtract `KERNBASE` and you're done. The `V2P` and `V2P_WO` macros defined in
[memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h) do just that, and the `P2V` and `P2V_WO` add `KERNBASE` to a
physical address to get the kernel virtual address.

Note that I said "kernel virtual address", not just any old virtual address.
Users don't get these kinds of fancy privileges, because they shouldn't be
worrying about where anything is in physical memory. They're running through a
limitless field of virtual memory, remember? So user virtual addresses between 0
and 2 GB will get mapped to totally arbitrary locations in physical memory.

One consequence of this is that xv6 is limited to no more than 2 GB of physical
memory (instead of the 4 GB that 32-bit addresses allow for) in order to map it
all into the top 2 GB of virtual memory. In reality, it's even less, for two
reasons: (1) we also need to map device I/O regions into virtual memory, so
it'll be a little less than 2 GB, and (2) it's hard and annoying to figure out
how much physical memory is actually present on any given machine, so xv6 just
says to hell with all that and picks the totally arbitrary value of a puny 224
MB as the amount of available physical memory (that's `PHYSTOP`, defined in
[memlayout.h](https://github.com/mit-pdos/xv6-public/blob/master/memlayout.h)).

## Paging

Remember when we talked about segmentation, and how we said we'd come back to
paging later? Guess what? It's later.

So all virtual addresses are really "logical addresses", and segmentation turns
those into "linear addresses". In xv6, the boot loader set up the segmentation
hardware to use an identity map, so virtual addresses are the same as logical
addresses are the same as linear addresses. Now paging has to turn those linear
addresses into physical addresses. Just like segmentation uses a GDT and the
segment registers for its mapping, paging uses a page directory, page tables,
and the `%cr3` register.

First, imagine a world where every single time some user code throws up an
address (maybe it looks up a variable, or it calls a function, or it simply
needs to execute the next instruction), the CPU has to stop what it's doing,
save all the user's register contents, load up some kernel code, restore its
register contents, find out where its stack is, get it running, and then ask the
OS where that virtual address is actually located in physical memory. That would
be *so* slow. We don't want that. We want the hardware to do all the address
conversions by itself, and involve the OS only minimally to set up a new page
directory when it starts a new process.

Instead, the x86 hardware uses one of its control registers, `%cr3`, to store a
pointer to a page directory in memory. Then every time it needs to map a linear
address to a physical one, it goes to that page directory and grabs the relevant
entry. That entry is a pointer to a page *table* somewhere else in memory, so
the processor grabs the right entry from there, which points to a 4096-byte page
in some other location.

A linear address has a three-part structure: the 10 most significant bits are an
index that picks an entry from the page directory, the next 10 bits are an index
to pick an entry from whatever page table we've been directed to, and the last
12 bits are an offset that determines where to look in the page that the page
table entry pointed to.

For example, let's say we have a virtual address like 0x9C4A_02BF. If we convert
to binary, split it up, and convert back to hex, we can see that the 10 most
significant bits are 0x271, the next 10 are 0x0A0, and the last 12 are 0x2BF. So
the paging hardware would look at wherever `%cr3` is pointing to find the page
directory; let's just call it `pgdir`. Then it would take entry `pgdir[0x271]`
and go look wherever that's pointing to find the right page table; let's call
that `pgtab271`. Then it would take entry `pgtab271[0x0A0]` and look wherever
that's pointing to find the right page, `pg`. *Then* it would finally
know that the corresponding physical address is `pg + 0x2BF`. Whew.

This still sounds super slow, so the paging hardware uses a cache called the
Translation Lookaside Buffer (TLB) to store recently-used mappings and make them
faster in the future. Since pages are 4096 bytes, it only needs to map a new
page if the addresses some code is asking for crosses a page boundary.

xv6 provides two macros, `PDX` and `PTX` defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h), to recover just the
page directory index bits or the page table index bits, respectively, from a
virtual address.

Finally: an important aspect of virtual memory is that each process should be
isolated from the others, and the kernel should be isolated from user processes.
So each process will get its own page directory, and each entry of that page
directory will say whether it's present (i.e., mapped) or not. If it's present,
then it points to a page table for that process; if it's not present and we try
to access it, we'll get a page fault or a general protection fault. Each entry
in a page table will also say whether that page is present and what kinds of
permissions it has. The bit flags for the permissions are (in order from least
to most significant bit):
* Bit 0: present.
* Bit 1: read/write.
* Bit 2: user (otherwise only the kernel can access it).
* Bit 3: write-through.
* Bit 4: cache disabled.
* Bit 5: accessed (for the TLB).
* Bit 6: page size (we'll talk about this later).
* Bit 7: (unused).

This way, since each process has its own page directory, page tables, and pages,
and each level has specific permissions set, they should never be able to
interfere with each other.

Again, most of the time, the kernel will just happily ignore all this and use
the mapping in the higher half of virtual memory for simplicity. Each user
process's page directory will have the same mapping in the higher half so that
the kernel can keep doing what it's doing no matter which user process is
currently running.

Anyway, back to the code! We left off after the boot loader had finished loading
the kernel into memory; it ended by calling an `entry()` function in the kernel.
We haven't set up paging yet, so that's next on our to-do list. But first, the
kernel is compiled and linked using a *linker script*, so we'll have to look at
that to understand how that sets up memory the way we want it.

## kernel.ld

The gory details of linker scripts as a whole are outside the scope of these
posts, so I'm gonna gloss over a lot of the parts of this file and focus on
the important pieces.

It's important to understand what a linker does in a rough sense, so I'll just
generalize and wave my hands around and say that a compiler takes code in a
high-level language and converts it to assembly, an assembler takes that
assembly code and turns it into machine code, and a linker takes a whole bunch
of machine code files (including any code for library functions) and links them
all together into a single executable file.

Linking involves three steps that are important for us here: first, the linker
has to assign each piece of code a location in memory, so that different
variables, functions, etc. don't end up colliding; then it replaces references
to that object with its address. Second, it has to resolve any outstanding
symbols (variables, functions, etc.) in each file by looking them up in all the
other files and replacing them with those addresses; the linker can define its
own symbols too. Third, it has to create an output file in a format that the OS
can use, like ELF.

xv6 has decided that command-line flags are too basic for it, so instead it'll
use a linker script [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld) for the GNU linker.

We start off by specifying the output format (32-bit ELF), the architecture
(x86, also known as i386), and the entry point to start executing code. The
convention is to call the entry point `_start`; the ELF header will include its
address, which is how we were able to call it from the boot loader.
```ld
OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(_start)
```

Next up come the sections. Remember the ELF sections `text`, `rodata`, `data`,
`bss`, and `stab`? Well we've gotta tell the linker where to set them up in
memory, using commands like `. = address`. These are virtual addresses, so since
we want to set up our kernel in the higher half of virtual memory, we'll tell it
to link the code start at 0x8010_0000. Again, we use that address instead of
0x8000_0000 (which maps to physical address 0) because we have to avoid the
address spaces of the boot loader and the memory-mapped I/O devices.

We can also tell the linker where in physical memory the code should be placed
(in linker script lingo, its "load address") using the `AT(address)` command.
We'll use the physical address 0x0010_0000, since that maps to virtual address
0x8010_0000.
```ld
SECTIONS {
    . = 0x80100000;

    .text : AT(0x100000) {
        /* this part tells the linker which files to include in this section */
    }

    /* more sections here... */
}
```

There's one other detail we should check out: the linker can create its own
symbols using the `PROVIDE(symbol = .)` command. If the code happens to declare
its own variable `symbol`, then the linker will just throw away its own version
of it, but if the code uses `symbol` without defining it, then the linker will
replace those references with the contents of that memory location.
```ld
SECTIONS {
    /* virtual address and text sections are defined as above */

    PROVIDE(etext = .);     /* etext will be at the address right after the end
                            of the text section */

    /* rodata, stab, and stabstr sections defined here */

    PROVIDE(data = .);      /* data will be at the address at the very beginning
                            of the data section */

    /* data section defined here */

    PROVIDE(edata = .);     /* edata will be at the address right after the end
                            of the data section */

    /* bss section defined here */

    PROVIDE(end = .);       /* end will be at the very last address at the end
                            of the entire kernel code */
}
```

Those variables will be used later in the kernel code; not so much for their
contents but for their addresses, as pointers to the virtual addresses of
specific parts of the kernel's code in memory. On to the kernel!

## entry.S

I have bad news. That `entry()` function that the boot loader called? It's in
assembly again. :(

### Multiboot Header

Okay, so first off, we've got some more hideous specs to deal with for a bit in
the form of a multiboot header. Multiboot is a specification that lets boot
loaders load up kernel code in a standardized way; the GNU boot loader GRUB uses
it. So this part is mostly here in case you want to run xv6 on real hardware
using GRUB; feel free to skip to `entry()` below.

The original Multiboot specification has since been replaced with Multiboot 2,
but again, it's 1995, so we don't know about that yet.

Multiboot helps compliant kernels and boot loaders identify each other using a
special header. The header must be completely contained in the first 8192 bytes
of the kernel's image, and it must be 32-bit aligned. The header contains three
things: (1) a magic number used for mutual identification and recognition
(0x1BADB002 for kernels, 0x2BADB002 for boot loaders), (2) some flags for the
kernel to inform the boot loader what the kernel requires in order to run
successfully, and (3) a 32-bit unsigned checksum which when added to the other
two fields must have a 32-bit unsigned sum of zero. Depending on the flags that
are set, there may be other components to the Multiboot header.

So we'll start by creating a `multiboot_header` label at the beginning of the
file (and thus, the beginning of the kernel image) and making sure it's aligned
to 32 bits.
```asm
.p2align 2      # Force 4-byte alignment
.text
.globl multiboot_header
multiboot_header:
    # ...
```

Now we'll just add the magic number, set the flags to 0 to indicate no special
requirements, and add the checksum.
```asm
    #define magic 0x1badboo2
    #define flags 0
    .long magic
    .long flags
    .long (-magic-flags)
```

And that's it!

### entry

Back in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), we said that the linker would set up the kernel's ELF
header to specify the kernel's entry point using `_start`, but `_start` itself
wasn't actually defined there, so we have to do that first. We don't know where
this code will end up in memory, so we'll define an `entry` label and set
`_start` to the address of `entry`. Note that the linker script used virtual
addresses in the higher half, but we haven't set up paging yet, so we'll have to
convert it to a physical address using one of the macros we mentioned earlier.
```asm
.globl _start
_start = V2P_WO(entry)
.globl entry
```

Next up we want to finish setting up virtual memory by enabling paging, but
that's all kinds of complicated, so we're gonna start off with a super simple
version of paging. Part of that difficulty is that there's a bootstrap problem:
we need to allocate pages to hold the page tables themselves, but we can't use
pages without page tables... uhh...

We'll solve that by starting off with a basic, super-simple page directory where
only two entries are mapped: the first entry maps virtual addresses 0 to 4 MB to
physical addresses 0 to 4 MB, and the second entry maps virtual addresses
`KERNBASE` to `KERNBASE` + 4MB to physical addresses 0 to 4 MB. One consequence
is that the entire kernel code and data has to fit in 4 MB.

Why the two entries pointing to the same place? It's to solve another bootstrap
problem. The kernel is currently running in physical addresses close to 0. Once
we enable paging and start using virtual addresses in the higher half, the stack
pointer `%esp`, instruction pointer `%eip`, even the pointer in `%cr3` to the
page directory itself will all still point to low addresses until we update
them. But updating them requires executing instructions, which would require
accessing low addresses a few more times. If we left out the low addresses, we'd
get a page fault, and since we don't have exception handlers set up yet, that
would cause a double fault, which would turn into the dreaded **TRIPLE FAULT**,
in which the processor enters an infinite reboot loop. So yeah, point is, we
need both the low and high mappings for now; we'll get rid of the low mappings
once we're done setting up.

But wait! Aren't page directory entries supposed to point to page tables? How
can they point directly to pages here? It turns out that x86 can skip that
second layer altogether if we use so-called "huge" pages of 4 MB in size instead
of the usual 4 KB. In the long run, this could lead to internal fragmentation,
but it does cut down on the overhead and allows a faster set-up. Plus we're only
gonna use them for a minute while we get ready for the full paging ordeal.

To use 4 MB pages, we have to enable x86's Page Size Extension (PSE) by setting
the fourth bit in the `%cr4` register. `CR4_PSE` is defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as 0x10,
or 00010000 in binary.
```asm
entry:
    movl    %cr4, %eax
    orl     $(CR4_PSE), %eax
    movl    %eax, %cr4
```

We need a page directory before we can set up paging; again, basic version now,
full glorious page directory later. We're gonna do the same thing we did in the
boot loader where we tell the processor to load the page directory now but then
procrastinate actually writing it; this time, we'll write it in C and call it
`entrypgdir`. Then we'll load its physical address into register `%cr3`.
```asm
    movl    $(V2P_WO(entrypgdir)), %eax
    movl    %eax, %cr3
```

Now we can enable (a basic version of) paging! We tell the CPU to start using
the page directory in `%cr3` by setting bit 31 (paging) of register `%cr0`; we
can also set bit 16 (write protect) of the same register to prevent writing to
any pages that the page directory and page tables have marked as read-only.
`CR0_PG` and `CR0_WP` are defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) to set these bits.
```asm
    movl    %cr0, %eax
    orl     $(CR0_PG|CR0_WP), %eax
    movl    %eax, %cr0
```

Now remember how the processor is still running at low addresses? Yeah, let's
fix that. First we'll make a new kernel stack in the higher half that will still
be valid even after we get rid of the lower address mappings. We'll have the
linker save some space for us under the symbol `stack` and set it up there;
`KSTACKSIZE` is defined in [param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 4096 bytes. So we just set the stack
pointer register `%esp` to the top of that section in order to let the stack
grow down toward the address of `stack`. Again, we'll procrastinate actually
defining `stack`.
```asm
    movl    $(stack + KSTACKSIZE), %esp
```

Now we want to call into the `main()` function, but we don't just want to do
that the usual assembly way of `call main`. That would generate a jump relative
to the current value of `%eip`, which is still in low addresses. We'll use an
indirect jump instead.
```asm
    mov     $main, %eax
    jmp     *%eax
```

Finally, we need to get around to reserving space for the stack. We can do that
with the assembler instruction `.comm symbol, size`:
```asm
.comm stack, KSTACKSIZE
```

## main.c

Awesome, back to C code now! Remember how we procrastinated actually defining
`entrypgdir`? Let's do that now; it's at the bottom of [main.c](https://github.com/mit-pdos/xv6-public/blob/master/main.c).

### entrypgdir
What in the world is this?!
```c
__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
    [0] = (0) | PTE_P | PTE_W | PTE_PS,
    [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```
Okay, bear with me; I promise it's not too bad.

First, the `__attribute__` tells the compiler and linker that the page directory
should be placed in memory at an address that's a multiple of `PGSIZE` (4096
bytes); that's just a requirement of the paging hardware.

Next, we define `entrypgdir` as an array of `NPDENTRIES` (1024, according to
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)), each of type `pde_t` (a type alias for `unsigned int`, according to
[types.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h)).

Then we initialize the entries: in C, you're allowed to initialize an array by
specifying the values of specific enties; all other enties become zero. You
specify an entry by putting its index in square brackets before its value, so
`[2] 5` will set the entry with index 2 to be 5. Here we initialize the entries
with indices 0 and `KERNBASE >> PDXSHIFT`, which is the same thing as
`PDX(KERNBASE)`, AKA the page directory index corresponding to the virtual
address `KERNBASE`, AKA 0x8000_0000. So basically, we've initialized the page
directory entries corresponding to the low virtual address 0 and the high
virtual address `KERNBASE`.

We set their value to 0, because we want them to map to physical addresses from
0 up to 4 MB. Oh, and remember how page directories and page tables can also
hold permission flags? We want to set flags to say that these pages are present
(so that accessing them doesn't cause a page fault), writeable, and 4 MB in
size; those are defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as `PTE_P`, `PTE_W`, and `PTE_PS`. We can
combine them all together by bitwise-ORing them.

And we're done!

### main

The code in [entry.S](https://github.com/mit-pdos/xv6-public/blob/master/entry.S) finished up by calling into the C function `main()`, which
is where the core set-up happens before we can start running processes. It calls
into basically every single part of the xv6 kernel, so we can't go through all
the functions line-by-line yet; instead I'll just give you an overview of what
they do.

* `kinit1()` solves another bootstrap problem around paging: we need to allocate
    pages in order to use the rest of memory, but we can't allocate those pages
    without first freeing the rest of memory, which requires allocating them...
    You see what I mean. This function will free the rest of memory between the
    `end` of the kernel code (defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), remember?) and 4 MB.
* `kvmalloc()` allocates a page of memory to hold the fancy full-fledged page
    directory, sets it up with mappings for the kernel's instructions and data,
    all of physical memory, and I/O space, then switches to that page directory
    (leaving poor old `entrypgdir` in the trash).
* `mpinit()` detects hardware components like additional CPUs, buses, interrupt
    controllers, etc. Then it determines whether this machine supports this
    crazy new idea where you can have multiple CPU cores. Wow, 1995 is crazy.
* `lapicinit()` programs this CPU's local interrupt controller so that it'll
    deliver timer interrupts, exceptions, etc. when we're ready for them later.
* `seginit()` sets up this CPU's kernel segment descriptors in its GDT; we still
    won't really use segmentation, but we'll at least use the permission bits.
* `picinit()` disables the *ancient* PIC interrupt controller that literally no
    one has ever used since the APIC was introduced in 1989. I don't even know
    what to say. I guess I was mistaken when I assumed it was 1995; I don't
    know.
* `ioapicinit()` programs the I/O interrupt controller to forward interrupts
    from the disk, keyboard, serial port, etc., when we're ready for them later.
    Each device will have to be set up to send its interrupts to the I/O APIC.
* `consoleinit()` initializes the console (display screen) by adding it to a
    table that maps device numbers to device functions, with entries for reading
    and writing to the console. It also sets up the keyboard to send interrupts
    to the I/O APIC.
* `uartinit()` initializes the serial port to send an interrupt if we ever
    receive any data over it. xv6 uses the serial port to communicate with
    emulators like QEMU and Bochs.
* `pinit()` initializes an empty process table so that we can start allocating
    slots in it to processes as we spin them up.
* `tvinit()` sets up and interrupt descriptor table (IDT) so that the CPU can
    find interrupt handler functions to deal with exceptions and interrupts when
    they come.
* `binit()` initializes the buffer cache, a linked list of buffers holding
    cached copies of disk data for more efficient reading and writing.
* `fileinit()` sets up the file table, a global array of all the open files in
    the system. There are other parts of the file system that need to be
    initialized like the logging layer and inode layer, but those might require
    sleeping, which we can only do from user mode, so we'll do that in the first
    user process we set up.
* `ideinit()` initializes the disk controller, checks whether the file system
    disk is present (because both the kernel and boot loader are on the boot
    disk, which is separate from the disk with user programs), and sets up disk
    interrupts.
* `startothers()` loads the entry code for all other CPUs (in [entryothers.S](https://github.com/mit-pdos/xv6-public/blob/master/entryothers.S))
    into memory, then runs the whole setup process again for each new CPU.
* `kinit2()` finishes initializing the page allocator by freeing memory between
    4 MB and `PHYSTOP`.
* `userinit()` creates the first user process, which will run the initialization
    steps that have to be done in user space before spinning up a shell.
* `mpmain()` loads the interrupt descriptor table into the CPU so that we're
    finally completely ready to receive interrupts, then calls the `scheduler()`
    function in [proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c), which enables interrupts on this CPU and starts
    scheduling processes to run. `scheduler()` never returns, so at that point
    we're completely done with setup and we're running the OS proper.

## Summary

The entry code in the xv6 kernel had one job: to set up paging. It kind of
failed at that job, but not for lack of trying! There are just all kinds of
Catch-22s when it comes to paging, so at least it got us partway there by making
a temporary page directory to tide us over until we can throw it away and never
look back.

We also took a sneak peek at all the setup code in `main()`; we're gonna end up
going through it all, but at least now you should have enough of an idea of
what's going on that you can more or less skip around and look at what you need.
