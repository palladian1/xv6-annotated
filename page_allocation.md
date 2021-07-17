# Page Allocation

When we left off before the lock detour, the boot loader had set up a GDT to
ignore segmentation, and the entry code set up some barebones paging with an
`entrypgdir`. But that initial page directory is too limiting to keep for long;
it only mapped the first 4 MB of physical memory. So we want a new one, but we
have to set it up and allocate pages in it before we can actually use it. And
until we switch to it, everything has to happen in those first 4 MB.

## kalloc.c

We start off in this file by declaring the function `freerange()`, which will be
defined below. We have to do this in C in order to call a function in the code
before the compiler has actually seen the function's definition, which comes
below, or maybe in another file. A *declaration* tells the C compiler "I know I
haven't shown you this symbol before, but don't worry; it's just a function that
takes this number of arguments with these types and has a return value of this
type." That lets the compiler keep calm and carry on with its usual type-checks
(weak as they may be in C). A *definition* tells the compiler that this is the
function (or variable) we were talking about, so it'll reserve some space in
memory for it; it also tells the compiler how to evaluate that function whenever
it's called (for variables, an *initialization* will have to tell the compiler
what the value the variable should hold). The linker will take care of matching
function calls (and variable uses) to their definitions, possibly across files.

Usually you'd stick declarations in a C header file and tell the preprocessor to
copy-paste the header into your code with an `#include` directive; then other
files could `#include` that header too. So header files should really be more of
an API kind of thing, for functions that you want other code to be able to call.
This one is just a local helper function, so we'll declare it here instead of in
a header so other code can't use it.
```c
void freerange(void *vstart, void *vend);
```

Okay okay, I know function declarations are like 101-level C, but I wanted to
mention them because we're about to see something similar but a little off next
when we declare `end` as a global array of characters.
```c
extern char end[];
```
The C keyword `extern` lets you define a global variable or function in one file
and use it in another, so in that sense it's similar to the function declaration
above. In fact, the compiler implicitly assumes there's an `extern` before each
function declaration. The difference is that an explicit `extern` lets us do the
same thing for global variables: we tell the compiler and linker "hey, I'm gonna
use a variable of this type with symbol `end`, but don't worry about reserving a
spot in memory for it; that already happened elsewhere."

The really cool thing about `extern` is that the function or variable might not
even be defined in C -- it could come from any other language! We just pass the
compiled object files from the other language together with the C object files
to the linker and it'll match up the definitions and calls.

In this case if you try looking for the place where `end` is defined in the C or
assembly code, you're gonna be disappointed. Turns out it's actually defined in
[kernel.ld](https://github.com/mit-pdos/xv6-public/blob/master/kernel.ld), remember? Back then, we said it was gonna be located at the very
first memory address right after the end of the kernel code and data in memory.
We're about to see why it's needed.

Next up, we define a new `struct` type:
```c
struct run {
    struct run *next;
};
```
Hmm, the only member of this `struct run` is a pointer to another `struct run`.
Hopefully, you've seen some singly-linked lists before so you can recognize it
as one of those. Usually it would have another member to hold the data in the
list, but we won't need any extra data here; we'll find out why soon enough.

Last thing before we get to the functions: we define another `struct` type and
declare the global variable `kmem` to be of that type.
```c
struct {
    struct spinlock lock;
    int use_lock;
    struct run *freelist;
} kmem;
```
The syntax here is the usual C thing where we say the type of a variable, then
an identifier, like `int i`; it just looks more confusing because we're also
defining the type at the same time. This `struct` type doesn't get a name like
`struct run` did because we're only gonna need it this one time. The fields are
a spin-lock (hence the detour before coming here), a `use_lock` variable that
we'll treat as a boolean, and a pointer to a `struct run` called `freelist`.

I'm just gonna go ahead and spoil the next two functions for you: we want to use
a better page directory than `entrypgdir`, right? Well then we need to assign
a page of memory for it, plus a page for each of its page tables, plus a page
for each entry in those page tables that's mapped. That means we'll need some
bookkeeping to track which pages have already been assigned. We're gonna use a
linked list of free pages (that's what `struct run` is for); we'll allocate a
page by popping one off the free list, and we'll free a page by pushing it onto
the top of the list.

Note that `kfree()` here is *not* supposed to be a kernel version of the usual C
standard library function `free()`, nor is `kalloc()` supposed to be a kernel
version of `malloc()`. We have no concept of a heap yet, so heap allocation
wouldn't make sense. These functions allocate and free *whole physical pages* to
be added to the current page directory and its page tables.

### kfree

This function will free a single page (4096 bytes, or `PGSIZE`) of memory by
adding it to the front of the free list. It takes an argument `char *v` which is
a virtual address; we're using `char *` here instead of `uint *` or `void *` or
whatever so that the pointer arithmetic increments by a single byte instead of
4 bytes for `uint` or whatever.

First, some sanity checks: `v` should be page-aligned (because we're freeing a
whole page), it should be above `end` (because we don't want to accidentally
overwrite the kernel code), and its corresponding physical address should be
below `PHYSTOP` (because the only addresses we'll use above the top of physical
memory are for memory-mapped I/O devices and we shouldn't be freeing those pages
anyway).
```c
void kfree(char *v)
{
    if ((uint) v % PGSIZE || v < end || V2P(v) >= PHYSTOP) {
        panic("kfree");
    }
    // ...
}
```

Now, if you've programmed in C, you might have come across the dreaded (but oh-
so-common) bug known as a *use-after-free*. This means you called `free()` on
some variable (hopefully one you had `malloc()`-ed before), and then used it
again. Hmm, very naughty! The problem is that that memory might have been re-
allocated to some other variable or even another process, so you might read the
wrong values or overwrite something important. This is a *very* common cause of
security vulnerabilities in C and C++ to this day; it's also not always easy to
spot because huge projects might have you call `malloc()` in one file, then use
the variable somewhere else thousands of lines of code later in some other file,
then call `free()` in yet another file -- plus it's unlikely that all of these
pieces were written by the same person. So let's make this a little easier on
ourselves by filling the freed page with junk (a bunch of 1s everywhere) in the
hope that a use-after-free leads to a crash (and thus debugging and detection)
sooner than it would otherwise.
```c
void kfree(char *v)
{
    // ...
    memset(v, 1, PGSIZE);
    // ...
}
```
You might be familiar with `memset()` from the C standard library in *string.h*,
but we can't risk using standard library functions here because they assume the
code will be provided by the OS, and the implementation might require any of a
million features we haven't implemented yet. So we have to make our own version
for the kernel in [string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c). We'll get around to looking at that code later on
in an optional detour, but for now just know that it sets the memory starting at
`v` and continuing for `PGSIZE` bytes to hold a bunch of repeated 1s.

Now let's talk concurrency. At any time, multiple threads might want to allocate
or free pages simultaneously; if we're not careful we might accidentally use the
same page twice, which would cause bugs in addition to security vulnerabilities,
because all the per-process isolation that paging gets us would be lost. So much
work down the drain! This is why `kmem` has a lock, which we should use any time
we push to or pop from the free list.

But in the early stages of the kernel we only use a single CPU and interrupts
are disabled, so there's nothing to fear. Plus, locks add overhead, and the
`acquire()` function needs to call `mycpu()`, which we haven't even defined yet,
so let's just go ahead and skip them in the beginning. So `kmem.use_lock` is a
boolean that will tell us whether we need a lock right now or not.
```c
void kfree(char *v)
{
    // ...
    if (kmem.use_lock) {
        acquire(&kmem.lock);
    }
    // ...
}
```

Okay, we're finally at the point where we can free the page. We'll make a
`struct run *r` that points to virtual address `v`, then make its `next` point
to the first entry of the free list. Then we'll update the head of the list to
point at the newly-freed page. This is the standard C idiom to add to the front
of a singly-linked list.
```c
void kfree(char *v)
{
    // ...
    struct run *r = (struct run *) v;
    r->next = kmem.freelist;
    kmem.freelist = r;
    // ...
}
```

There's something interesting here: where are we storing this entry for the free
list? Why, in the free page itself! So each unused page will hold the address of
the next one in its first few bytes.

Finally, we're out of the critical section where we updated the free list, so we
can release the lock.
```c
void kfree(char *v)
{
    // ...
    if (kmem.use_lock) {
        release(&kmem.lock);
    }
}
```

### kalloc

Allocating a page means popping off the head of the free list. We acquire the
lock first, if we need one.
```c
char *kalloc(void)
{
    if (kmem.use_lock) {
        acquire(&kmem.lock);
    }
    // ...
}
```

Next, we get a pointer to the first free page in the list and update the head to
point to the next one in the list. But what if the list is empty? In that case,
the head would be a null pointer, and dereferencing a null pointer (like we do
here in `r->next`) is undefined behavior in C, which means BAD THINGS HAPPEN.
I'm serious -- there are absolutely no restrictions on what might happen, so the
compiler could literally set your computer on fire if it wanted to. In the real
world, that usually means either a segmentation fault or security vulnerability,
or both if you're unlucky. So we should check whether `r` is null (i.e. zero).
if it's nonzero then we can update `r->next`; otherwise we should just return
`r` and hope whoever called us checks whether it's null. Moral of the story:
any call to `kalloc()`, just like any call to `malloc()` in regular C code,
should always be followed by checking whether the returned pointer is null.
```c
char *kalloc(void)
{
    // ...
    struct run *r = kmem.freelist;
    if (r) {
        kmem.freelist = r->next;
    }
    // ...
}
```

Okay, so now we just release the lock, and we're done!
```c
char *kalloc(void)
{
    // ...
    if (kmem.use_lock) {
        release(&kmem.lock);
    }
}
```

### freerange

`kalloc()` and `kfree()` both handle only one page at a time, which can get
annoying if we're trying to free tons of pages at once; also, they can only use
page-aligned virtual addresses, which have to be typecast to `char *`. Let's
simplify our lives with a simple wrapper function to free multiple pages between
two virtual memory addresses `vstart` and `vend` that may not be page-aligned.

Let's assume that `vstart` is the first address after some other data in an
already-allocated page; we don't want to free that page, but the next one, so we
align it to a page boundary by rounding up, then cast that to a `char *`.
```c
void freerange(void *vstart, void *vend)
{
    char *p = (char *) PGROUNDUP((uint) vstart);
    // ...
}
```

Now we can iterate over the pages, starting at `p` and incrementing by `PGSIZE`
until we reach or pass `vend`, freeing pages as we go.
```c
void freerange(void *vstart, void *vend)
{
    // ...
    for (; p + PGSIZE <= (char *) vend; p += PGSIZE) {
        kfree(p);
    }
}
```
Done, next.

### kinit1 and kinit2

Both of these functions get called by the kernel's `main()`. Quick reminder:
we've got an `entrypgdir` that maps two virtual address ranges (0 to 4 MB and
`KERNBASE` to `KERNBASE` + 4 MB) to the physical addresses range from 0 to 4 MB.
We want to leave this baby page directory behind for a grown-up page directory
that maps all of physical memory, but first we needed to figure out how to
allocate pages.

Okay cool, we already did that. But allocation needs a free list, which for now
is just sitting around chilling as an empty list. But we can't free pages if
they're not already allocated, right? Ahh, bootstrap problems! This one's not an
issue; we'll just cheat this one time and free all the memory between `end` (the
end of the kernel code and data in memory) and `PHYSTOP`, even though we didn't
get it from a call to `kalloc()`. Sounds good, right?

I hate to burst your bubble, but kernel development *loves* bursting bubbles.
Turns out there's yet another bootstrap problem: each page has to store the
pointer to the next free page, which means we have to write to that page, which
means that page must already be mapped... but we can't map all of memory until
we initialize the free list by freeing all of memory...

HEAD. DESK. We're screwed.

Okay, obviously the xv6 authors figured this out already. The trick is that we
do have *some* physical memory we can write to: everything between `end` and 4
MB. So we can free that part for now, allocate some of those pages for a fresh
page directory and some pages, then use those pages to map the rest of physical
memory, then come back later and free those pages.

So we'll have to split up the work of setting up the new page directory into two
very similar functions, `kinit1()` and `kinit2()`. The first one will initialize
the lock for the free list but make `kmem.use_lock` false so we don't use a lock
in the early stages of kernel setup. The second one will set it to true so we
start using a lock to allocate and free pages once we have multiple CPUs, a
scheduler, interrupts, etc.

Both of them will use `freerange()` to free the pages in a section of physical
memory. `main()` calls `kinit1()` with arguments to free the range from `end` to
4 MB, and calls `kinit2()` with arguments for the range from 4 MB to `PHYSTOP`.
```c
void kinit1(void *vstart, void *vend)
{
    initlock(&kmem.lock, "kmem");
    kmem.use_lock = 0;
    freerange(vstart, vend);
}

void kinit2(void *vstart, void *vend)
{
    freerange(vstart, vend);
    kmem.use_lock = 1;
}
```

## Summary

This whole file was just to set up page allocation for the new page directory
we're gonna replace `entrypgdir` with. It uses a free list in `kmem`; freeing a
page adds it to the front of the list and allocation pops a page off the front.
We have to populate the free list will pages for all of physical memory, but we
do that in two steps to avoid some bootstrap issues.

Again, this is a *page* allocator, not a *heap* allocator like `malloc()`, but
many heap allocator implementations use linked lists of free heap regions in the
same way. We talked about use-after-free bugs above, but now we can also see why
*double-frees* (in which you free the same memory region more than once) can
cause bugs and security vulnerabilities: they add the same region to it twice,
which then might get allocated to two different variables or processes, which
might ruin the per-process isolation that virtualization is supposed to provide.
In addition, our page allocator handles fixed-size regions, but a heap allocator
needs to use variable regions, so when a memory region gets allocated twice
after a double-free, it might get split up into differently-sized pieces, of
which some parts get allocated to other processes, etc... It's just a nightmare.

Next up, we'll see the full story of virtual memory.
