# More Paging: The Kernel Side

We've already talked *plenty* about virtual memory, and I bet you're probably so
over `entrypgdir` by now; let's wrap up its story and get rid of it!

The [vm.c](https://github.com/mit-pdos/xv6-public/blob/master/vm.c) file is HUGE; only [proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c) and [sysfile.c](https://github.com/mit-pdos/xv6-public/blob/master/sysfile.c) match its length. Some
parts deal with the general paging implementation; we'll look at those here. The
rest handles the details of paging for processes and user code, we'll need to
know a bit more about processes in xv6 for that.

## vm.c

After the include directives for the preprocessor, we have a declaraction for
an external symbol defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld). This one is the beginning of the
data section for the kernel.
```c
extern char data[];
```

Next we have a definition for a pointer to a global page directory: this is the
fancy new one that's gonna replace `entrypgdir`. Note that `pde_t` is a type for
page directory entries defined in [types.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h); it's just a type alias for `int`.
```c
pde_t *kpgdir;
```

### seginit

This first function gets called directly by the kernel's `main()`; it sets up
the segment descriptors in the GDT as identity maps to all of memory so that we
can ignore them from now on. Wait, didn't we already do that in the boot loader?

Yes, kind of, but that was before the kernel took over, so back then we had no
notion of kernel space versus user space. Now that we do, we want to set the
permission flags for each segment so that we can use the privilege ring levels,
with the kernel in ring 0 and user code in ring 3. That way any misbehaving user
code will get slapped with a segmentation fault the way we've all come to know
and love in C.

We also have some permission flags for protection in the page directory and page
table entries, so maybe we could get away without it? I mean, both kernel code
and user code are read-only anyway, so maybe they could both have a Descriptor
Privilege Level of 3. But no, x86 is gonna shut that right down by forbidding
interrupts that take you from ring level 0 to ring level 3, so all the interrupt
handler functions have to be in kernel space with a kernel code segment selector
at ring level 0.

So we're just gonna have to do it all over again. Great. Well, maybe it's not
too bad, let's take a look... oh god, it's awful. Okay, deep breath.

Each processor has its own GDT, so we're gonna need to call this function once
per CPU. First we figure out which CPU we're on with with the `cpuid()` function
that we'll see later on; for now it... (drumroll)... gets the CPU's ID. Then we
look that up in a global table of CPUs (there's an `extern` declaration for this
in the included [proc.h](https://github.com/mit-pdos/xv6-public/blob/master/proc.h)) and store it in a `struct cpu`; we saw that before in
the spin-lock code, but we'll get around to talking about it more later.
```c
void seginit(void)
{
    struct cpu *c = &cpus[cpuid()];
    // ...
}
```

That `struct cpu` has a field to hold the GDT, so we're gonna add entries for
the kernel code, kernel data, user code, and user data segment descriptors;
those entries are `SEG_KCODE`, `SEG_KDATA`, `SEG_UCODE`, and `SEG_UDATA`,
respectively. Recall that the permission bits are `STA_X` (executable), `STA_R`
(readable), and `STA_W` (writeable); now we're gonna pile on the descriptor
privilege levels for the kernel (0) and user (3, or `DPL_USER`) on top. Besides
those ring levels, we want to ignore segmentation, so each segment should be an
identity map for all virtual memory from 0 to 4 GB (0xffff_ffff).
```c
void seginit(void)
{
    // ...
    c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
    c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
    c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
    c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);
    // ...
}
```
The only difference between the `SEG` macro used here and the `SEG_ASM` one from
the boot loader is that this one is for C code and the other is for assembly.

Finally, we load up the new GDT into the processor with a C wrapper for the
x86 instruction `lgdt`.
```c
void seginit(void)
{
    // ...
    lgdt(c->gdt, sizeof(c->gdt));
}
```

Done with segmentation, now on to more paging.

### walkpgdir

A page directory lets the paging hardware convert virtual addresses to physical
ones, but we're gonna need those mappings in the kernel too while we set up the
page directory, so this function does the conversion manually. Wait, but aren't
we setting up paging so that all of physical memory is mapped in the higher half
of the virtual address space? Can't we just add or subtract `KERNBASE` to do the
conversion? Well, that would work for kernel virtual addresses, but user virtual
addresses actually will use page directories and page tables in a non-obvious
way, so if we want to figure out where those go, we'll need a function for it.

In C, using the `static` keyword before a function limits its scope and makes it
visible only within its own file. The function returns a `pte_t *`, a pointer to
a page table entry (the type is defined in [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) as a type alias for `uint`).

Its arguments are a pointer to a page directory, a virtual address, and `alloc`
(a boolean variable, but as an `int` instead of `bool`). This `alloc` lets the
function play a dual role: if it's set, the function will allocate a page table
if needed; otherwise it reports failure if the page table doesn't exist. The
`const` keyword lets the compiler know a variable shouldn't be mutated so it'll
throw an error if we do. Here, `const void *va` is a pointer to a constant value
of any type; the address the pointer holds might change, but we can never write
to that address. The opposite is a `void *const va`: the address being pointed
to will never change, but we can overwrite the contents of that address all we
want. You can combine the two with `const void *const va`. What's that I hear? C
syntax is the worst? No, never...
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
}
```

Remember way back when, when we talked about how "linear" addresses are set up
and converted to physical ones? The first 10 bits are an index for the page
directory to pick a page directory entry, which points to a page table; the next
10 bits pick a page table entry that points to a page, and the last 12 bits are
an offset within that page; the `PDX()` and `PTX()` macros get first 10 bits and
the next 10 bits from a linear address, respectively. So we start by getting the
page directory index and using that to get the page directory entry.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    pde_t *pde = &pgdir[PDX(va)];
    // ...
}
```

Okay, so now `pde` points to a page directory entry which has two parts: a
pointer to the physical address of a page table, and some flags. But who knows
if this page table even exists; most page directory (and page table) entries
aren't mapped in order to save space. So we have to check whether `*pde` has the
`PTE_P` (present) flag set.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
    }
    // ...
}
```

If the page table exists, we should get rid of the flags and recover the pointer
to the page table using the `PTE_ADDR()` macro. But the hardware uses physical
addresses for these pointers, so we need to convert it to a virtual address
first, which is what this function does... recursion? Bootstrap problem? No,
it's actually easy because we can access the page table from within the kernel's
virtual address space in the higher half by adding `KERNBASE` to the physical
address with the `P2V()` macro.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        pgtab = (pte_t *) P2V(PTE_ADDR(*pde));
    } else {
        // ...
    }
    // ...
}
```

Now for the else clause, which happens if the page directory entry doesn't have
the `PTE_P` bit set. Well, if the boolean `alloc` is false (zero), then we're
done and we should just report failure by returning a null pointer. On the other
hand, if it's true, we just allocate a page for the page table. But wait,
remember how page allocation might fail and return a null pointer if we're out
of free pages in the free list? And remember how I said we should always check
for that? Okay well let's check for that; if allocation fails, we also return a
null pointer. Oh, and because this is C, we're gonna do a jillion things at once
in a single line: check if `alloc` is false, try to allocate a page table, and
check if that allocation failed. C lets us assign to a variable and then test
that variable's value in a single statement.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        if (!alloc || (pgtab = (pte_t *) kalloc()) == 0) {
            return 0;
        }
        // ...
    }
    // ...
}
```

Okay, so now suppose: (1) the page table wasn't present, (2) alloc was set, and
(3) we successfully allocated a page. Now what? Remember how we filled all free
pages with garbage in `kfree()` using `memset()`? Let's undo that now by zeroing
it.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
        memset(pgtab, 0, PGSIZE);
        // ...
    }
    // ...
}
```

Now we'll update the page directory entry to point to this new page table and
add the `PTE_P` flag so it knows it's present. Wait, while we're at it, what
other permissions will it need? Is it writeable? Can users access it? Hmm, we'd
have to know whether we're looking up a user virtual address or a kernel one,
and whether it's gonna be used for code or data. Ah, screw it, we'll just throw
all the flags on there at once. Either way, the page table entries will have
their own flags too, so we can restrict the page's permissions there instead of
here at the page directory entry.
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    pte_t *pgtab;
    if (*pde & PTE_P) {
        // ...
    } else {
        // ...
        *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
    }
    // ...
}
```
This probably isn't the safest thing ever, because we're saying that only the
page table will restrict permissions, so we're throwing all that responsibility
over there, but hey, xv6 is supposed to be simple, not ultra-secure. Just don't
do this at home, kids.

Finally, we return the address of the corresponding page table entry using the
index from the middle bits of `va`:
```c
static pte_t *walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
    // ...
    return &pgtab[PTX(va)];
}
```

### mappages

Okay, so `walkpgdir()` returns a pointer to a page table entry and can even
crate a page table if if it doesn't exist. That's not quite enough to add new
mappings for pages though; the page itself might not be mapped, and if we just
created a new page table, then certainly none of the pages are mapped yet.
`mappages()` will finish the job by installing mappings in page tables (possibly
newly-allocated ones) for a range of virtual addresses.

The arguments are a page directory, a virtual address for the beginning of the
range, the size of the range, a physical address to map it to, and the flags for
permissions we want to set. We start off by rounding the virtual address down to
the nearest page boundary and getting a pointer to the end of the range, also
page-aligned.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    char *a = (char *) PGROUNDDOWN((uint) va);
    char *last = (char *) PGROUNDDOWN(((uint) va) + size - 1);
    // ...
}
```

Now we're gonna iterate over the pages in that range; `for (;;)` is a common C
idiom for an infinite loop. In this case, we need to increment `a` and `pa` by
`PGSIZE` each time, and we'll break out of the loop when `a` reaches `last`. To
be completely honest, I'm not really sure why the authors chose to write this as
an infinite loop with the condition/break statement and update statements inside
the loop rather than as a regular old for loop; I think the latter would be more
clear, but oh well, I didn't write this.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        if (a == last) {
            break;
        }
        a += PGSIZE;
        pa += PGSIZE;
    }
    // ...
}
```

Inside the for loop, we'll start each iteration by looking up the right page
table entry with `walkpgdir()`, with `alloc` set to true. Remember how that
function called `kalloc()`, which might fail, in which case it returns a null
pointer? Well that means we've gotta check for a null pointer here too. This
time however, we'll return -1 for failure and 0 for success, because why not?
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, a, 1)) == 0) {
            return -1;
        }
        // ...
    }
    return 0;
}
```

We're supposed to be allocating brand-new pages for this range of addresses, so
if a page has already been allocated, we'll just flip out in rage and panic.
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        if (*pte & PTE_P) {
            panic("remap");
        }
        // ...
    }
    // ...
}
```

The last thing before checking the loop condition and updating `a` and `pa` is
to install the mapping to the right physical address with the right permissions
in the page table. Then we're done!
```c
static int mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
    // ...
    for (;;) {
        // ...
        *pte = pa | perm | PTE_P;
        // ...
    }
    // ...
}
```

Cool, now we have a way to map new pages into a page directory. We're well on
our way to leaving poor old `entrypgdir` behind for the shiny new `kpgdir`.

### kmap

Each process is gonna have its own page directory, so its mappings in the lower
half of the virtual address space might be totally different from those of
another process. But the mappings in the higher half (where the kernel lives)
will always be the same -- that way, the kernel can always use the existing page
directory for whatever process it happens to be running. We'll only use `kpgdir`
when the kernel isn't currently running a process, e.g. while it's running the
scheduler.

So when we create a new process, we'll need to copy in all the mappings that the
kernel expects to find into a fresh page directory for that process. Those are:
memory-mapped I/O device space from physical address 0 to 0x10_0000 (the boot
loader is also here, but we don't need it any more), kernel code and read-only
data from 0x10_0000 to the physical address of `data` (one of the symbols
defined in [kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld)), kernel data the rest of physical memory from there to
`PHYSTOP`, and more I/O devices from 0xFE00_0000 and up. Each of these ranges
needs its own permissions too.

We'll represent each of these mappings with a `struct kmap`, which has fields
for the starting virtual address, the starting and ending physical addresses,
and the permissions; then the mappings will get stored in a static global
variable `kmap`... oh come on, what fresh hell is THIS?
```c
static struct kmap {
    void *virt;
    uint phys_start;
    uint phys_end;
    int perm;
} kmap[] = {
    { (void *)KERNBASE, 0, EXTMEM, PTE_W },
    { (void *)KERNLINK, V2P(KERNLINK), V2P(data), 0 },
    { (void *)data, V2P(data), PHYSTOP, PTE_W },
    { (void *)DEVSPACE, DEVSPACE, 0, PTE_W }.
};
```

Okay, there are a few things going on here. First, the `static` keyword for a
variable means that variable has a single fixed location in memory that it's
never gonna move out of.

Then it does that thing again where we simultaneously define a `struct` type and
define a variable of that type. So the type is
```c
struct kmap {
    void *virt;
    uint phys_start;
    uint phys_end;
    int perm;
};
```

So then the static global variable `kmap` is an array of `struct kmap`s. I guess
we ran out of names or something. The array has four entries, and since each one
is a `struct`, it needs curly braces around it.

The first entry (for the lower of the two memory-mapped I/O device regions) has
a `virt` field of `KERNBASE`, a `phys_start` field of 0, a `phys_end` field of
`EXTMEM` (defined as 0x10_0000), and permission flag `PTE_W`. So it maps a
virtual address range starting at `KERNBASE` to the physical address range from
0x0 to 0x10_0000 and makes it writeable so we can communicate with the devices
there. The next two entries are similar, except that the kernel code isn't
writeable.

The last entry has `phys_start` of 0xFE00_0000 and a `phys_end` of 0. That's a
little strange, but it's because we want to map all the way up to the end of the
virtual address space at 0xFFFF_FFFF. The end should be one byte past that, but
it's impossible to represent 0x1_0000_0000 with 32 bits. Setting the end to 0
makes the size calculation (`phys_end - phys_start`) work out nicely: it'll just
overflow to the right number. This is okay since we're using unsigned integers,
but note that *signed* integer overflow is undefined behavior and thus VERY BAD
and the cause of many security vulnerabilities.

Okay, back to getting rid of `entrypgdir`!

### setupkvm

This function sets up a fresh new page directory with all the mappings in `kmap`
in order to please the kernel when it encounters the page directory. So needy,
right?

It takes no arguments and returns a pointer to the new page directory. First,
let's allocate a page of memory to hold the new directory. We'll be good and
remember to check for null (in which case we return null too) and clear the page
of the garbage values we wrote when we freed it.
```c
pde_t *setupkvm(void)
{
    pde_t *pgdir;
    if ((pgdir = (pde_t *) kalloc()) == 0) {
        return 0;
    }
    memset(pgdir, 0, PGSIZE);
    // ...
}
```

The upper end of virtual memory after `DEVSPACE` has I/O devices, so `PHYSTOP`
should be below that; this is as good a place as any to make sure.
```c
pde_t *setupkvm(void)
{
    // ...
    if (P2V(PHYSTOP) > (void *) DEVSPACE) {
        panic("PHYSTOP too high");
    }
    // ...
}
```

Finally, we'll add all the mappings in `kmap` above into this page directory so
the kernel is happy. We'll use `mappages()`, which returns -1 if it fails, so
we should check for that. The `freevm()` function is defined below, and we'll
get to it soon, but for now just know that it gets rid of all the mappings we
just made, in case any of them fails.
```c
pde_t *setupkvm(void)
{
    // ...
    struct kmap *k;
    for (k = kmap; k < &kmap[NELEM(kmap)]; k++) {
        if (mappages(pgdir,
                k->virt,
                k->phys_end - k->phys_start,
                (uint) k->phys_start,
                k->perm) < 0) {
            freevm(pgdir);
            return 0;
        }
    }
    return pgdir;
}
```
Let's check out that for loop: `k` is a pointer to a `struct kmap`, and `kmap`
is an array of `struct kmap`s; in C, arrays decay to pointers, so they have the
same type. `k` starts off pointing to the first (zero) entry of `kmap`. Then
incrementing it with `k++` shifts its value by the size of a `struct kmap`, so
it'll point to the next entry. The loop stops when `k` points beyond the last
entry of `kmap`, as determined by the `NELEM()` macro which counts the number of
entries in an array. Note that array element-counting only works in C if the
array is defined in the same function or as a global variable in the same file,
which is why it's so easy to do an out-of-bounds read or write in C (yet another
common security vulnerability).

Finally, if everything worked out okay, we return a pointer to the new page
directory.

### switchkvm

We said above that the kernel would usually just use the page directory of the
currently-running process, but it'll use `kpgdir` when no process is running,
i.e. during the kernel setup and while it's scheduling a new process. So we need
a way to tell the paging hardware to load `kpgdir` into register `%cr3`, which
holds a pointer to the page directory. That's this function.

It's a one-liner: get the physical address of `kpgdir` and stick it in `%cr3`
with the assembly instruction `lcr3`.
```c
void switchkvm(void)
{
    lcr3(V2P(kpgdir));
}
```

### kvmalloc

FINALLY, we're here! We're gonna get rid of `entrypgdir`! The kernel's `main()`
calls this function right after `kinit1()`.

We already did all the hard work, so this one's a breeze: we call `setupkvm()`
to allocate a new page directory and fill it with the kernel's mappings, then
call `switchkvm()` to load it into the paging hardware.
```c
void kvmalloc(void)
{
    kpgdir = setupkvm();
    switchkvm();
}
```

And we're DONE! Take that, `entrypgdir`, we don't need you anymore. We're big
kids now.

## Summary

So far, it's been a serious odyssey just to move from no paging in the boot
loader, to super basic paging with `entrypgdir` in [entry.S](https://github.com/mit-pdos/xv6-public/blob/master/entry.S), to `kpgdir` now.
Along the way, we've looked at code to allocate and free pages and install new
mappings in page directories and page tables. That'll come in handy when we look
at processes next; the virtual memory story still isn't over.

Also, note that `kpgdir` still isn't at the height of its powers: at the point
when `main()` calls `kvmalloc()`, the free list only contains pages for physical
memory between 0 and 4 MB. The rest will have to wait until `kinit2()` unleashes
its full potential. (Maybe some self-actualization seminars would help...)
