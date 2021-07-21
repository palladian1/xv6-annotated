# More Paging: The User Side

It's almost time to turn to interrupts and processes so we can figure out how to
work that sweet multiprocessing magic, but unfortunately we have some last
pieces of paging to wrap up before we can get there.

I know, we've been talking about virtual memory for what feels like a century
now, but so far everything we've done has been on the kernel side, allocating
pages and creating new page directories with the same kernel mapping. But what
about the lower half of the virtual address space, where user processes live?

This post will go through the rest of
[vm.c](https://github.com/mit-pdos/xv6-public/blob/master/vm.c)
and set up the paging-related machinery we'll need to run processes later on.

## Detour: Starting a New Process

When xv6 runs a new process, it will create a brand new virtual memory space for
it with a fresh page directory. We haven't talked about processes in xv6 yet, so
you might wonder how a process gets started up in the first place.

Let's forget all about xv6 for a second and think about another Unix-like OS:
Linux. How do we start a process there? Okay, we also have to forget about GUI
applications there. Let's just say you want to run some C code (xv6 maybe?) that
you've just compiled; what happens when you run it from the terminal?

Hopefully, you've done the OSTEP project called [processes-shell](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/processes-shell)
by now, so you know the answer; if you haven't, I recommend doing that one right
now before I give it away. (It's not strictly required, but are you really the
kind of person who loves getting movies spoiled for them?)

Okay, are you done?

The answer: it's just an `exec()` system call! The shell finds the executable
file in the file system, calls `fork()` to create a new child process, which
then calls `exec()` to transform itself into the program you want to run.

We'll get to these system calls later, so for now let's just go over the broad
strokes as they relate to virtual memory. `fork()` works by taking the parent
process's virtual memory space and making a copy of it for the child process.

`exec()` allocates a new page directory, figures out how much memory the new
program will need when it runs, then grows the virtual memory space allocated in
that new page directory to the required size. Then it loads the program into
memory in the new page directory.

Next, `exec()` skips a page, leaving it mapped but user-inaccessible; then the
next page becomes the process's stack. Why that empty page? It's an important
one for protection: that way, user programs that blow their stack will trigger a
page fault or a general protection fault instead of possibly overwriting random
code.

Then `exec()` copies some arguments into the stack before it switches to using
the new page directory and gets rid of the old one it had before.

Whew, okay, that's a lot of code to go over later, and that's only the virtual
memory part of the story. So let's just make it easier by doing all the work we
can right now. According to the above, we have to understand how xv6 does all of
the following:
* Makes a copy of a whole page directory,
* Creates a new page directory,
* Grows (or shrinks) the virtual memory space of a page directory,
* Loads program code into a page directory,
* Makes a page inaccessible to users,
* Copies stuff into a page in a page directory,
* Switches to a new process page directory, and
* Gets rid of an unused page directory.

Finally, there's one edge case to think about: running the very first process.
We obviously need to start running a shell at some point, so we need a special
way to get that started too, so it can in turn run other processes.

## vm.c, Again

We're gonna need some new functions! Actually, we already finished one of the
requirements -- `setupkvm()` can allocate a new page directory and set up the
kernel portion too. `switchkvm()` lets us switch to using `kpgdir` as a page
directory, but now we need to switch *away* from that to a page directory for a
process, so that'll be `switchuvm()`.

`copyuvm()` creates a copy of an entire page directory for a child process.
`allocuvm()` and `deallocuvm()` grow and shrink the virtual memory space that's
allocated in a page directory, and `freevm()` clears a page directory we no
longer need.

`loaduvm()` will load program code into a page directory; `clearpteu` makes a
page inaccessible to users, and `copyout()` copies data into a page in a page
directory. `inituvm()` handles the special case of setting up the page directory
for the very first process that xv6 will run.

The rest of this post will go over those functions one by one so we can be done
with virtual memory, but I know it's a little strange to go through a million
helper functions when we haven't seen the code that's gonna use them yet, so if
you'd prefer, you can come back to this after reading about processes and system
calls.

### deallocuvm

The arguments for this function are a page directory, the process's old size,
and the new size we want to shrink it down to; it'll return the process's new
size. By "shrinking" a virtual memory space, we really mean making sure that the
page directory only allocates up to `newsz` worth of pages. So if we think of
the sizes as virtual addresses, then the page directory currently maps the
virtual space from 0 to `oldsz`, so we should free everything between `newsz`
and `oldsz`, leaving behind the space from 0 to `newsz`.

First, we should make sure the new size is actually smaller than the old one;
otherwise trying to "shrink" down to the new size might cause integer overflow.
There; the sizes are both unsigned integers here, so at least it wouldn't be
that scary boogeyman of undefined behavior, but it could still be bad: 0 would
wrap around to 2^32 - 1, so "shrinking" to the new size would actually grow the
process way beyond what physical memory could handle.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    if (newsz >= oldsz) {
        return oldsz;
    }
    // ...
}
```

We're gonna shrink the physical memory allocated to this page directory by
freeing pages until we reach the new size. Let's start with the first page above
`newsz`; we can get its virtual address by rounding up `newsz` to a page
boundary.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    uint a = PGROUNDUP(newsz);
    // ...
}
```

Now we'll just iterate over the pages between `a` and `oldsz` one at a time and
free them. This is a little tricky: `kfree()` takes a virtual address (cast to a
`char *`), but it should be a *kernel* virtual address in the higher half, not a
user virtual address. Luckily, we already have `walkpgdir()`, which can take an
arbitrary virtual address and return its page table entry, so that's a good
start.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldz; a += PGSIZE) {
        pte_t *pte = walkpgdir(pgdir, (char *) a, 0);
        // ...
    }
    // ...
}
```

The page table entry contains the page's physical address, plus some flags to
determine whether it's mapped and what permissions are set for it.

Now, a virtual address space isn't laid out contiguously. Think about it: if you
sit back and imagine a user process hanging out in memory, what does that
address space look like? You're probably imagining the stack at one end of
memory and the heap at the other, with each growing toward the center, right?
so there will be some pages in the center that aren't mapped; some of the page
tables might not exist either, in which case `walkpgdir()` would return a null
pointer.

Remember we agreed to never dereference null pointers? Yeah, so we'll have to
skip all those unmapped pages. If we got a null pointer, then that means the
entire page table doesn't exist, so we need to skip forward to the next page
directory entry (and thus the next page table). We'll have to move `a` to the
virtual address that corresponds to that next page directory entry.

We can get the page directory index from `a` with the `PDX()` macro we've seen
seen before, and then just add 1 to get the next entry in the page directory.
Now we need to turn that back into a virtual address. We'll use a new macro,
`PGADDR()` (also from [mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)),
to do that. So then we'll continue to the next loop iteration, which will get
the page table entry for this new virtual address.

Wait wait wait, one last thing! After all that, `a` should now be the first
virtual address in the page table for the new page directory entry... except
it's get `PGSIZE` added to it because of the for loop's update statement.

Ugh, okay, fine, this is annoying. Let's just fix it with a hack: subtract
`PGSIZE` from it now, so that it gets incremented to the right value in the next
iteration. Okay, that's it, I swear!
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        pte_t *pte = walkpgdir(pgdir, (char *) a, 0);

        if (!pte) {
            a = PGADDR(PDX(a) + 1, 0, 0) - PGSIZE;
        } else {
            // ...
        }
    }
    // ...
}
```

Okay, now the else branch: if we don't get a null pointer then at least the page
table exists, but that doesn't mean the page itself is mapped. If it's not, then
we don't need to do anything else, but if it is mapped, then we need to free it.
We can get the page's physical address out of the page table entry with the
`PTE_ADDR` macro then make sure it's not null.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        // ...
        if (!pte) {
            // ...
        } else if ((*pte & PTE_P) != 0) {
            uint pa = PTE_ADDR(*pte);
            if (pa == 0) {
                panic("kfree");
            }
            // ...
        }
    }
    // ...
}
```

The whole point of this was to be able to call `kfree()`, remember? So let's
convert `pa` to a kernel virtual address as a `char *` and free it. Then after
the loop is done, we'll return the new size.
```c
int deallocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < oldsz; a += PGSIZE) {
        // ...
        if (!pte) {
            // ...
        } else if ((*pte & PTE_P) != 0) {
            // ...
            char *v = P2V(pa);
            kfree(v);
            *pte = 0;
        }
    }
    return newsz;
}
```

### allocuvm

This is the reverse of `deallocuvm()`: instead of freeing pages with `kfree()`,
we'll allocate them with `kalloc()`. Here too, we start by checking for integer
overflow by making sure `newsz` really is larger than `oldsz`. But now we also
have to check that we're not gonna grow the process's size into the region where
it could access kernel memory; otherwise it might read or modify arbitrary
physical memory.
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    if (newsz >= KERNBASE) {
        return 0;
    }
    if (newsz < oldsz) {
        return oldsz;
    }
    // ...
}
```

We're gonna start adding new pages right after `oldsz`, so we have to align that
to a page boundary:
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    uint a = PGROUNDUP(oldsz);
    // ...
}
```

The for loop is easier this time around because we already know that the pages
aren't mapped. First we allocate a new page. Any call to `kalloc()` needs two
things after, remember? We have to check for null, in which case we print an
error message to the console (that's `cprintf()`; we'll get to that in the
devices section), then undo any allocations we made and return 0. Then we have
to zero the page because we filled it with 1s when it was freed.
```c
int allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
    // ...
    for (; a < newsz; a += PGSIZE) {
        char *mem = kalloc();
        if (mem == 0) {
            cprintf("allocuvm out of memory\n");
            deallocuvm(pgdir, newsz, oldsz);
            return 0;
        }

        memset(mem, 0, PGSIZE);
        // ...
    }
    // ...
}
```

We have a page now, but it's not yet mapped in the page directory. We can do
that with `mappages()`; that might fail too (because it needs to allocate more
pages for the page tables), in which case we do the same as before. Then after
the for loop is done, we return the new size.
```c
int allocuvm(pde_t *pgdir), uint oldsz, uint newsz) {
    // ...
    for (; a < newsz; a += PGSIZE) {
        // ...

        if (mappages(pgdir, (char *) a, PGSIZE, V2P(mem), PTE_W | PTE_U) < 0) {
            cprintf("allocuvm out of memory (2)\n");
            deallocuvm(pgdir, newsz, oldsz);
            kfree(mem);
            return 0;
        }
    }
    return newsz;
}
```

### freevm

This function will get rid of a user page directory that we no longer need. Now
that we have `deallocuvm()`, it's easy: we just "shrink" the process to a size
of zero. Oh and we'll remember the lessons our ancestors have taught us and make
sure the pointer to the page directory isn't null before dereferencing it.
```c
void freevm(pde_t *pgdir)
{
    if (pgdir == 0) {
        panic("freevm: no pgdir");
    }
    deallocuvm(pgdir, KERNBASE, 0);
    // ...
}
```

Great, so all pages are freed, and we're done!

Now hang on a sec... The page directory itself resides in memory; so do the page
tables. We have to free those too. We'll start with the page tables; freeing the
page directory first would be a use-after-free vulnerability because we'd need
to use it to get to the page tables.

We'll iterate over the page directory's entries, checking whether each one has
the "present" flag set (`NPDENTRIES` is defined as 1024 in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/types.h)).
If it does, we'll get the page table's physical address from it with the
`PTE_ADDR()` macro, then convert that to a virtual address as a `char *` to make
`kfree()` happy. We don't have to worry about clearing the "present" flag in the
page directory because it's about to be freed anyway.
```c
void freevm(pde_t *pgdir)
{
    // ...
    for (uint i = 0; i < NPDENTRIES; i++) {
        if (pgdir[i] & PTE_P) {
            char *v = P2V(PTE_ADDR(pgdir[i]));
            kfree(v);
        }
    }
    // ...
}
```

We wrap up by freeing the page directory itself.
```c
void freevm(pde_t *pgdir)
{
    // ...
    kfree((char *) pgdir);
}
```

### copyuvm

The `fork()` system call will need to "clone" a process, which includes its
virtual address space. This function takes a pointer to the parent process's
page directory and the size of the parent process's address space and returns a
pointer to a fresh new page directory with everything set up exactly the same.

We start by creating a new page directory and taking care of the kernel's half
of the address space with `setupkvm()`. That might fail if it can't allocate a
new page, so we have to check for null. Sigh. C code is approximately 40%
checking for null return values.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    pde_t *d;
    if ((d = setupkvm()) == 0) {
        return 0;
    }
    // ...
}
```

Now we'll iterate over the user portion of the parent process's address space
from 0 to `sz`, copying everything over as we go. Say we want to copy a page
from the parent's virtual address `i` to the child's address `i` (note that
they'll map to different physical addresses). We'll have to figure out the
corresponding kernel virtual address for the parent's `i` in order to do that,
so we use `walkpgdir()` to get the page table entry, then get the page's
physical address.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, (void *) i, 0)) == 0) {
            panic("copyuvm: pte should exist");
        }
        if (!(*pte & PTE_P)) {
            panic("copyuvm: page not present");
        }

        uint pa = PTE_ADDR(*pte);
        uint flags = PTE_FLAGS(*pte);
        // ...
    }
    // ...
}
```
In this case we know the parent process is already set up, so we don't really
have to worry about `walkpgdir()` failing and returning null, but it's bad C
juju to ignore a possibly-null return value, so we just panic if it does fail or
if the page isn't present.

Next we allocate a page for the child process (checking for null again...) and
copy everything from the parent's page to the new child page.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        char *mem;
        if ((mem = kalloc()) == 0) {
            goto bad;
        }
        memmove(mem, (char *) P2V(pa), PGSIZE);
        // ...
    }
    // ...
}
```
You might recognize `memmove()` as a C standard library function that copies the
contents of one memory address into another, but we can't use those, remember?
So xv6 provides its own implementation of it in
[string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c).

If you haven't seen a `goto` statement before, it's basically a holdover from ye
olde days before Edsger Dijkstra preached the gospel of structured programming
to the world and invented the if statement. It does exactly what it sounds like:
you make a label somewhere in code and it takes you there.

Next we stick that new page into the child's page directory, checking for null
again. If `mappages()` fails, then the new page won't be in the page directory,
so we have to free it here or else we'll never be able to find it again: a
memory leak.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        if (mappages(d, (void *) i, PGSIZE, V2P(mem), flags) < 0) {
            kfree(mem);
            goto bad;
        }
    }
    // ...
}
```

If none of the allocations failed, we just return a pointer to the new page
directory. But if something went wrong, then one of those `goto` statements will
send us to the time out corner of `bad`, where we undo all our work by freeing
the page directory and returning a null pointer.
```c
pde_t *copyuvm(pde_t *pgdir, uint sz)
{
    // ...
    return d;

bad:
    freevm(d);
    return 0;
}
```
Great, another function we'll have to check for null.

### switchuvm

Okay, we've got a way to create a new process page directory. We also have a way
to switch to using the kernel page directory `kpgdir` with `switchkvm()`. But we
need a way to switch to using the process page directory too. Enter `switchuvm()`.

I'll warn you -- `switchkvm()` was nice and short, but `switchuvm()` is an ugly
one for sure.

The argument to this function is a pointer to a `struct proc`, which represents
a process. We'll talk about that more when we get to processes; two fields are
important now: `p->kstack` which holds a pointer to the kernel stack for that
process, and `p->pgdir`, which points to that process's page directory.

Okay, well let's start with some sanity checks to make sure that the process `p`
actually exists (the pointer is non-null) and its kernel stack and page directory
pointers are non-null too.
```c
void switchuvm(struct proc *p)
{
    if (p == 0) {
        panic("switchuvm: no process");
    }
    if (p->kstack == 0) {
        panic("switchuvm: no kstack");
    }
    if (p->kstack == 0) {
        panic("switchuvm: no pgdir");
    }
    // ...
}
```

The main function of loading the process's page directory will be the same as in
`switchkvm()`: just an `lcr3` instruction. But the difference now is that the
x86 architecture requires some additional bookkeeping for processes.

See, when the kernel runs a new process, the CPU will start executing different
instructions. But it needs a way to keep track of where it left off in the
kernel code so that it can pick the thread back up after the process is done
executing. Similarly, interrupts and system calls might change the running
process, so the CPU needs to record some metadata about the process's state too
before switching to another one. x86 does that by means of a structure called a
*Task State Segment*, or TSS.

The TSS holds information like the current state of certain registers (e.g.,
`%esp`, `%eip`, `%cr3`, etc.), segment descriptors (`%cs`, `%ss`, `%ds`, etc.),
the current privilege leve, and I/O privilege levels -- in other words, the
process's *context*. It can be located anywhere in memory, but the processor
needs to find it, so it uses an entry in the GDT called the TSS segment
descriptor that points to the TSS. Remember the GDT from way back when we were
talking about segmentation? Good times. The CPU holds a pointer to the GDT's TSS
entry in a special register called the task register.

Back in the segmentation days of our youth, we stored the GDT in a `struct cpu`
that held information about the current processor. We got that `struct cpu` by
calling a `mycpu()` function. We're gonna do the same thing here in order to
update the GDT with a segment for the TSS. Getting interrupted in the middle of
this might be disastrous: the TSS would be half-updated, so who knows what would
happen when the CPU tried to resume execution where it last left off. So we'll
use the `pushcli()` and `popcli()` functions we saw with spin-locks to temporarily
disable interrupts.
```c
void switchuvm(struct proc *p)
{
    // ...
    pushcli();

    mycpu()->gdt[SEG_TSS] = SEG16(STS_T32A, &mycpu()->ts, sizeof(mycpu()->ts)-1, 0);
    mycpu()->gdt[SEG_TSS].s = 0;
    // ...

    popcli();
}
```
Whoa okay what is this?

We've seen the `SEG()` and `SEG_ASM()` macros before; they created GDT segments.
`SEG16()` does the same with 16 bits (it's defined in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h)). `STS_T32A`
is a flag that sets the segment's type as an available 32-bit TSS. Then we pass
in a pointer to the task state with `&mycpu()->ts`, its size, and a descriptor
privilege level of 0 (which means ring 0, the kernel level). The GDT's `.s`
field is a one-bit flag to determine whether this is a system or application
segment, so we set it to system.

Okay, so now the GDT points to the task state. Next we need to update the task
state, then load it into the CPU. We'll start by storing a segment selector and
the stack pointer in the task state; these should look familiar from the boot
loader and `seginit()`.
```c
void switchuvm(struct proc *p)
{
    // ...
    mycpu()->ts.ss0 = SEG_KDATA << 3;
    mycpu()->ts.esp0 = (uint) p->kstack + KSTACKSIZE;
    // ...
}
```

The TSS can also specify permissions for accessing I/O ports: for example,
setting the I/O privilege level to 0 in the `eflags` register *and* setting a
part of the TSS called the I/O map base address to an address beyond the TSS
segment forbids I/O instructions like `inb` and `outb` from user space. So we'll
set the I/O map base address next.
```c
void switchuvm(struct proc *p)
{
    // ...
    mycpu()->ts.iomb = (ushort) 0xFFFF;
    // ...
}
```

So now we have a GDT entry pointing to the TSS, which is now updated. Now we
just load it into the task register with the x86 instruction `ltr`; here we use
a C wrapper for that assembly instruction, defined in
[x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h).
```c
void switchuvm(struct proc *p)
{
    // ...
    ltr(SEG_TSS << 3);
    // ...
}
```

Finally, the last thing we do before re-enabling interrupts is to load the
process's page directory into the `%cr3` register so we can start using it.
```c
void switchuvm(struct proc *p)
{
    // ...
    lcr3(V2P(p->pgdir));
    // ...
}
```

### loaduvm

Okay, this is another function that's gonna require extra info we haven't seen
yet, but I'm gonna make it a bit easier by waving my hands around and glossing
over the details. It's gonna read a program from a file into memory at virtual
address `addr` using page directory `pgdir`. The part we want to read has size
`sz` and is located at position `offset` within the file.

Now, what about the file? We'll talk more when we get to the file system code,
but for now let's just say that files are represented in xv6 as `struct inode`s,
and we can read from them with the function `readi()`.

We're gonna run the program from this code, so the address it's stored in needs
to be page-aligned.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    if ((uint) addr % PGSIZE != 0) {
        panic("loaduvm: addr must be page aligned");
    }
    // ...
}
```

Next we're gonna iterate over pages starting from `addr`, reading from the file
in `ip` into that page. As usual, we'll need to get the kernel virtual address
from the user address `addr`, so we start by getting the page table entry via
`walkpgdir()`, checking for a null pointer if the corresponding page table
doesn't exist. Then we can turn that into a physical address.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // Get the page table entry
        pte_t *pte;
        if ((pte = walkpgdir(pgdir, addr + i, 0)) == 0) {
            panic("loaduvm: address should exist");
        }
        // Get the page's physical address
        uint pa = PTE_ADDR(*pte);

        // ...
    }
    // ...
}
```

Now we want to read from the file one page at a time using `readi()`, which
takes a pointer to an inode (here, `ip`), a kernel virtual address (`P2V(pa)`),
the location within the file of the segment we want to read (`offset + i`), and
the segment's size.

Now we want to read from the file one page at a time using `readi()`. We have to
specify a size in bytes to read; if the remaining unread part of the segment is
larger than a page, then the size we pass to `readi()` should be `PGSIZE`, but
otherwise it'll be less. So we'll compare `sz` to `i` and define define `n`
accordingly.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        uint n;
        if (sz - i < PGSIZE) {
            n = sz - i;
        } else {
            n = PGSIZE;
        }
        // ...
    }
    // ...
}
```

The other arguments to `readi()` are a pointer to an inode (`ip`), a kernel
virtual address (`P2V(pa)`), and the location within the file of the segment we
want to read (`offset + i`). It returns the number of bytes read, so if it's not
`n` we'll report an error by returning -1. Otherwise we return 0 after the for
loop is done.
```c
int loaduvm(pde_t *pgdir, char *addr, struct inode *ip, uint offset, uint sz)
{
    // ...
    for (uint i = 0; i < sz; i += PGSIZE) {
        // ...
        if (readi(ip, P2V(pa), offset + i, n) != n) {
            return -1;
        }
    }
    return 0;
}
```

### inituvm

Okay, the next three are nice and easy! This next one is pretty similar to
`loaduvm()`, except instead of loading program code from disk, it copies it in
from memory. We'll take `sz` bytes from a source address of `init` and stick it
in address 0 of the process's page directory `pgdir`.

This function is also easier because we're only gonna call it for programs that
are less than one page in size, so we don't have to worry about looping over
pages or anything like that. I like it when xv6 keeps things simple.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    if (sz >= PGSIZE) {
        panic("inituvm: more than a page");
    }
    // ...
}
```

Next we allocate a fresh page of memory, zero it to clear the garbage values,
and stick it into `pgdir` at address 0.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    // ...
    char *mem = kalloc();
    memset(mem, 0, PGSIZE);
    mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W | PTE_U);
    // ...
}
```

And we wrap up by actually loading the code from `init` into the new page.
```c
void inituvm(pde_t *pgdir, char *init, uint sz)
{
    // ...
    memmove(mem, init, sz);
}
```

### clearpteu

This function takes a page directory and a user virtual address and clears the
"user-accessible" flag so that the process can't touch it. It's used to create
an inaccessible page below a new process's stack to guard against stack
overflows; this way, a stack overflow will cause a page fault instead of
silently overwriting memory.

The `PTE_U` flag is in the page table entry, so we'll have to get that, then set
the flag.
```c
void clearpteu(pde_t *pgdir, char *uva)
{
    // Get the page table entry
    pte_t *pte = walkpgdir(pgdir, uva, 0);
    if (pte == 0) {
        panic("clearpteu");
    }
    // Clear the user permission flag
    *pte &= ~PTE_U;
}
```
Here `&` is a bitwise-AND and `~` is a bitwise-NOT; for reference, `|` is
bitwise-OR and `^` is bitwise-XOR. Contrast these with their logical versions,
`&&`, `!`, and `||` (XOR has no logical version). C also has corresponding
assignment operators (similar to `+=`, `-=`, `*=`, etc.) for each of them. So
the last line of code is equivalent to `*pte = *pte & (~PTE_U)`.

### uva2ka

We often need to convert user virtual addresses to kernel ones; `uva2ka()` is a
short helper function that does that while checking that the page is actually
present and has the user permission flag set.

We'll call walkpgdir to get the page table entry, then check both permission
bits before recovering the page address with `PTE_ADDR()` and converting it to a
kernel virtual address. We'll return the kernel virtual address as a `char *`,
or null if either flag is not set.
```c
char *uva2ka(pde_t *pgdir, char *uva)
{
    pte_t *pte = walkpgdir(pgdir, uva, 0);

    if ((*pte & PTE_P) == 0) {  // check that it's present
        return 0;
    }
    if ((*pte & PTE_U) == 0) {  // check that it's user-accessible
        return 0;
    }
    return (char *) P2V(PTE_ADDR(*pte));
}
```

Let me ask you a weird question: how are you feeling right now?

Okay, that was a test of your C coding practices, because if you took those null
checks to heart, you should be *really* uncomfortable right about now.

Check it out: `walkpgdir()` returns a pointer to the page table entry. *Any*
time a function returns a pointer, you should immediately ask yourself whether
that function can return a null pointer. Tons of C functions report an error by
returning null. In this case, we *know* `walkpgdir()` can fail and report null
if the page table doesn't exist, so we *know* we might get a null pointer out of
it -- it'll happen whenever a page table doesn't exist. So what do we do with
that knowledge?

Why, we go right ahead and dereference that pointer. WKBW;NQ39Q2A4T8YHMFGRW!!!

Dereferencing a null pointer is undefined behavior. There's literally no telling
what might happen. It can cause all kinds of bugs from segmentation faults to
security vulnerabilities.

All those null checks in the other functions serve a purpose: if something goes
wrong and a function returns a null pointer, they catch it before it gets
dereferenced, then either handle it gracefully or simply propagate the error by
returning null (or some other error code) and let the caller figure out what to
do with it.

Omitting a check for a null pointer like `uva2ka()` does is bad practice in C
because it means the programmer has to *guarantee* -- by manually checking --
that no call to this function could *ever possibly* cause a null return value.
Except humans are dumb, dumb creatures who make mistakes all the time, especially
in big projects: there's no way you'd be able to remember that tiny little
detail two years later when you decide to refactor your code or add a new
feature or something.

But maybe you can note that in the comments? Okay yeah, but think about it: how
often do you go and look up the source code for every single function you call?
Yeah, I thought so.

This is why C is so dangerous: there are hundreds of such problems that you need
to be aware of and remember to add stuff like null pointer checks to your code.
If you don't because you're a normal human who forgets things sometimes, then
you'll need to remember that you forgot to do it before and manually check every
single call to your code and think about every possible edge case that a
malicious adversary might exploit.

Good thing no one ever makes these mistakes in C, or we'd see enormous security
vulnerabilities being reported every single day in all kinds of critical
software. Oh wait...

So if you ever find yourself looking at C during code review and you come across
a function that returns a pointer, you should stop what you're doing and look up
the documentation for that function. If that function has any chance of
returning a null pointer, then you should yell and kick and scream until somebody
adds a null check and figures out how they want to handle it if it's null. Is
this annoying? Yes. Hard to remember? Yes. But that's C. *(cough cough use Rust
instead cough cough...)*

Now, the xv6 authors are so awesome that I'm gonna give them the benefit of the
doubt and assume they left it off because they hand-checked every call to make
sure it would never be an issue. But you and me? Nah.

The point of my rant is this: if you're reading this, then you're probably gonna
find yourself hacking away at xv6 for a project sooner or later. When you do
that, you should treat this function as VERBOTEN. You're not allowed to touch
it or call it, at least until you add a null check to it yourself.

The same goes for any functions that call this one, because maybe all the
existing calls to `uva2ka()` are fine right now, but then you make some tiny
change and now it's no longer guaranteed to never be null. For reference, this
function currently only gets called by `copyout()`, and that one only gets
called by `exec()`. `exec()` gets called by `sys_exec()`, the shell, and the
initial user-space program `init`. So be careful if you touch any of those.

Whew, okay, /rant.

### copyout

This function copies `len` bytes of data from a kernel virtual address `p` to a
user virtual address `va` using page directory `pgdir`. `exec()` will use this
to copy command-line arguments to the stack for a program it's about to run.

You might be wondering why it's needed -- doesn't `memmove()` do the same thing?
Almost, but the difficulty is that `pgdir` may not be the current page
directory, so we'll have to manually translate the virtual address `va`. That's
where `uva2ka()` comes in, plus it ensures that the page for `va` has the right
flags set. *Then* we can use `memmove()`.

First, `p` will be the source address, but `memmove()` requires a `char *` in
order to copy data byte-by-byte, so let's convert it now:
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    char *buf = (char *) p;
    // ...
}
```

Next we need to get the kernel virtual address corresponding to `va`, but
there's a challenge: what if the data crosses a page table boundary? It might be
spread across separate locations in physical memory (and thus in kernel virtual
memory too). So we'll need a loop in which each iteration gets the next kernel
virtual address and copies whatever part of the data is in this page.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        len -= n;
        buf += n;
        // ...
    }
    // ...
}
```

So we'll start each iteration by making `va0` the base address of the page `va`
is on and `pa0` the kernel address of `va0`, converted with `uva2ka()`. I...
honestly don't know why they used `pa0` as an identifier here. It makes it look
like it should be a physical address, but it's not; it's a kernel virtual
address. Sigh. Anyway, the call to `uva2ka()` might fail if the page isn't
present or it doesn't have a user permission bit, so we have to check for a null
pointer and return -1 if we find one.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        uint va0 = (uint) PGROUNDDOWN(va);

        char *pa0 = uva2ka(pgdir, (char *) va0);
        if (pa0 == 0) {
            return -1;
        }

        // ...
        va = va0 + PGSIZE;
    }
    // ...
}
```

Now `va` is in between `va0` and the next page, so the length of the data within
this page is `PGSIZE - (va - va0)`, unless it's the last page, in which case we
should pick the lesser of this value and `len` (since `len` gets decremented on
each iteration through the loop).
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        uint n = PGSIZE - (va - va0);
        if (n > len) {
            n = len;
        }
        // ...
    }
    // ...
}
```

Finally, we copy the data from `buf` into the target kernel virtual address for
`va`. Hmm, we don't have that yet. Oh wait, `pa0` is the kernel virtual address
for `va0`, and `va` is just `va-va0` bytes after that, so we'll use it.
```c
int copyout(pde_t *pgdir, uint va, void *p, uint len)
{
    // ...
    while (len > 0) {
        // ...
        memmove(pa0 + (va - va0), buf, n);
        // ...
    }
    return 0;
}
```
We return 0 if everything went okay.

## Summary

Okay, that was a lot of helper functions, but we're ALL DONE with virtual
memory! From now on, we have all the tools we'll need to manage memory and set
up virtual address spaces for new processes.
