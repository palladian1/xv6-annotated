# It's a Trap!

The last post introduced the mechanisms that xv6 uses for scheduling and context
switches. User processes can transfer control to kernel code with system calls,
potentially switching into the scheduler with `sleep()` or `exit()` to find
another process to run. But there are many other system calls besides those two
Kernel code can also be invoked during hardware interrupts or software
exceptions; these three together are collectively referred to as traps.

We'll go over traps now to understand them more generally. First, about the
terminology: depending on the source, interrupts might mean hardware interrupts
specifically or any trap generally; similarly, exceptions might mean errors
arising from the code, or traps in general. It's super frustrating because it
makes it really hard to know what's meant by a word like "interrupt" or
"exception" in whatever specification or source you happen to be reading. So I'm
gonna try my best to save you that kind of pain in this post by sticking to
"interrupt" for the hardware interrupts only, "exception" for software errors,
and "trap" for those two combined with system calls.

## Interrupt Descriptor Table

Imagine if, after every single time some user code carried out a division, the
processor stopped, context switched into the kernel, and asked the kernel to
check if there was a division by zero and handle it if necessary. Or every time
a hardware interrupt happened, the kernel had to start polling all the devices
to figure out which one just yelled. No. Just no. Running kernel code for all
this would be way too slow.

So it's the processor that will have to detect traps and decide how to handle
them. But what exactly it should do for a specific trap depends on all kinds of
of particulars about that OS, e.g. a disk saying it's done reading from a file
might require updating some file system data or storing the disk data in a
specific buffer or something. That's too much responsibility for the processor.

Okay, so the kernel will set up a bunch of handler functions for every possible
type of trap. Then it tells the hardware, "Okay, so if you get a disk interrupt,
here are my instructions to handle that. For timer interrupts, use these
instructions. If a process tries to access an invalid page, do this..."
From then on, the processor can handle the traps without further input from the
kernel by looking up the interrupt number in a big table to get the trap handler
function that the kernel set up, then just running it.

In the x86 architecture, that table is called the *interrupt descriptor table*
or IDT. I know, I'm sorry, I promised I'd say "trap" for the general case, but
the x86 specs give it the official name of IDT even though it handles all the
traps. Sigh. It has 256 entries (so that's the maximum number of distinct traps
we can define); each one specifies a segment descriptor (ugh segmentation again,
you know what that means: opaque code) and an instruction pointer (`%eip`) that
tell the processor where it can find the corresponding trap handler
function.

xv6 won't use all 256 entries; it'll mostly use trap numbers 0-31 (software
exceptions), 32-63 (hardware interrupts), and 64 (system calls), all defined in
[traps.h](https://github.com/mit-pdos/xv6-public/blob/master/traps.h).
But we do have to stick all 256 in the IDT anway, so we're the unlucky fools who
get to write 256 functions' worth of assembly code by hand. Nah, just kidding:
xv6 uses a script in a high-level language to do that for us and spit out the
entries into an assembly file.

Unfortunately for us, that high-level language is Perl. Sigh. Perl is infamous
as a "write-only" language, so I guess instead we're just the unlucky fools who
get to try reading Perl.

## vectors.pl

Okay, I'm not gonna assume you know Perl, and either way I really don't wanna go
over every single line of this file. The syntax is similar enough to C's (except
that somehow they managed to make it even *worse* than C), so you can read it on
your own if you want.

Now, no script will be able to generate 256 completely unique assembly functions
with enough detail to handle each trap correctly, so each function in the script
has to be pretty generic. They're all gonna call the same assembly helper
function, which will call a C function where we can more comfortably code up
how to handle each interrupt.

The gist of this Perl script is that it prints a bunch of stuff using a for loop
with 256 iterations. The xv6
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile)
will run it from the command line with `./vectors.pl > vectors.S` so that the
output gets saved in an assembly file, which will then get assembled together
with all the other kernel code in `OBJS`.

The resulting assembly file will look like this:
```asm
.globl alltraps

.globl vector0
vector0:
    pushl   $0
    pushl   $0
    jmp     alltraps

.globl vector1
vector1:
    pushl   $0
    pushl   $1
    jmp     alltraps

.globl vector2
vector2:
    pushl   $0
    pushl   $2
    jmp     alltraps

# ...
```

Except that a handful of entries (8, 10 through 14, and 17) will skip one line
(I'll explain why below):
```asm
# ...

.globl vector8
vector8:
    pushl   $8
    jmp     alltraps

# ...
```

Then at the end, it defines an array `vectors` with each of those entries above:
```asm
# ...

.data
.globl vectors

vectors:
    .long vector0
    .long vector1
    .long vector2
    # ...
```

Okay, so those are all the handler functions; the `vectors` array holds a
pointer to each one. They're all more or less the same: most of them push zero
onto the stack, then all they push a *trap number* to indicate which trap
just happened, and then they jump to a point in the code called `alltraps`;
that's the assembly helper function I mentioned earlier.

A handful of the entries don't push zero on the stack: these are trap numbers
8 (a double fault, which happens when the processor encounters an error while
handling another trap), 10 (an invalid task state segment), 11 (segment
not present), 12 (a stack exception), 13 (a general protection fault), 14 (a
page fault), and 17 (an alignment check). These are special because the
processor will actually push an error code on the stack before calling into the
corresponding handler function in `vectors`. It doesn't push any error codes on
the stack for the others, so we just push 0 ourselves to make them all match up.

## trapasm.S

### alltraps

The processor needs to run the trap handler in kernel mode, which means we have
to save some state for the process that's currently running so we can return to
it later (similar to the `struct context` we saw before), then set things up to
run in kernel mode. The `alltraps` routine does just that.

Remember how we said the IDT holds segment selectors for `%cs` and `%ss`, plus
and instruction pointer `%eip`? (I know we haven't seen the code to create the
IDT and store the entries of `vectors` in it yet; we'll get to that below.) The
processor will start using those segments (and save the old ones) before running
the trap handler function. Each trap handler function in `vectors` above pushed
an error code (or 0) followed by a trap number. Now we have to push all the
other segment selectors on the stack one at a time, then push all the general-
purpose registers at once with the x86 instruction `pushal`.
```asm
.globl alltraps
alltraps:
    pushl   %ds
    pushl   %es
    pushl   %fs
    pushl   %gs
    pushal

    # ...
```

Cool, all the registers are saved now. So now we'll set up the `%ds` and `%es`
registers for kernel mode (`%cs` and `%ss` were already done by the processor).
```asm
    # ...
    movw    $(SEG_KDATA<<3), %ax
    movw    %ax, %ds
    movw    %ax, %es
    # ...
```

Now we're ready to call the C function `trap()` that's gonna do most of the
work. That function expects a single argument: a pointer to the process's saved
register contents. Well, we just pushed them all on the stack, so we can just
use `%esp` as that pointer.
```asm
    # ...
    pushl   %esp
    call    trap
    # ...
```

That function will return back here when it's done, so let's ignore the return
value by moving the stack pointer just above it (essentially popping it off the
stack).
```asm
    # ...
    addl    $4, %esp
    # ...
```

### trapret

We've talked about this function before; when we create a new process, it starts
executing in `forkret()`, which then returns into `trapret()`. More generally,
any call to `trap()` will return here as well.

This function just restores everything back to where it was before, popping
stored registers off the stack in reverse order. We can skip the trap number and
error code; we won't need them anymore. Then we use the `iret` or "interrupt
return" (though you should read that as "trap return") instruction to close out,
return to user mode, and start executing the process's instructions again.
```asm
.globl trapret
trapret:
    popal
    popl    %gs
    popl    %fs
    popl    %es
    popl    %ds
    addl    $0x8, %esp  # skip the trap number and error code
    iret
```

## trap.c

Okay, on to the main part of the code! We have to do two things here: stick the
trap handler functions in `vectors` into an IDT, and figure out what to do with
each interrupt type.

At the top, we've got four global variables. The IDT is represented as an array
of `struct gatedesc`s, defined in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h). It's worth
taking a look at because it uses an obscure C feature (bit fields); we'll do that
in the next section.

Then we declare the `vectors` array of trap handler (with an `extern` keyword,
since it's defined in an assembly file), a global counter `ticks` that tracks
the number of timer interrupts so far (basically a rough timer), and a lock to
use with `ticks`.
```c
struct gatedesc idt[256];
extern uint vectors[];
struct spinlock tickslock;
uint ticks;
// ...
```

### Bit Fields

This section will get *deep* into the weeds, so feel free to skip it if you're
having a nice day and don't want to spoil it by reading about a bunch of C
standards.

So far, we've used bit flags with regular integers by manually doing some bit
arithmetic to set one bit at a time. For example, the flags for page table and
page directory entries are defined as powers of 2 (e.g., `PTE_P` is 0x1, `PTE_W`
is 0x2, `PTE_U` is 0x4, etc.) so that we can set a specific bit using a bitwise-
OR like `pte |= PTE_U` or test whether it's set with a bitwise-AND like
`pte & PTE_P`.

But sometimes that can get annoying and hard to keep track of; wouldn't it be
nice if we could just have variables that represent a single bit? Or two bits,
or any number of bits we want?

The trouble is that most computer architectures don't work with a single bit at
a time; they operate on bytes, words (2 bytes), long/double words (4 bytes), or
quad words (8 bytes), so it would be nontrivial to compile a line of C like
`a = 1` if `a` is a nonstandard size.

In fact, accessing variables that aren't aligned to a standard size (4 bytes on
x86 or 8 bytes on x86_64) is much slower than when they are aligned. Compilers
often optimize code to correct for this by padding `struct`s so that they'll
line up along those standard sizes. For example, one like
```c
struct nopadding {
    int n;
};
```
is probably left the same on x86, but one like this:
```c
struct padding {
    char a;
    int n;
    char b;
};
```
is probably converted by the compiler into this:
```c
struct padding {
    char a;
    char pad0[3];
    int n;
    char b;
    char pad1[3];
};
```

WARNING: We're entering the dark arts of C's unspecified and implementation-
defined behavior here. Note that these are different from *undefined* behavior:
undefined behavior means you did something BAD BAD BAD like dereferencing a null
pointer, freeing a memory region twice, using a variable after freeing it,
accessing an out-of-bounds index in a buffer, or overflowing a signed data type.
Implementation-defined and unspecified behavior aren't as dangerous as undefined
behavior is, but they can cause portability issues.

The C standard is a huge document with a bunch of legalese rules about what
makes C, well, C. People who write C compilers need to know exactly how C code
should behave under all kinds of different circumstances, so the C standard
spells most of it out. But there are some parts it intentionally leaves out.

*Implementation-defined* behavior means the C standard doesn't set any fixed
requirements about how a compiler should handle some behavior or feature; the
developers of a C compiler get to decide how to write that part of the code with
total freedom. One example is the number of bits in a byte; we've been assuming
it's 8, but there are some (dumb) architectures where it's different.

*Unspecified behavior*, on the other hand, means that the C Standard provides
some specific options, and compiler developers have to choose from those options
for *each instance* of the behavior in the code they're compiling (that means,
don't assume it's always gonna be the same, even with the same compiler).

Structure padding is implementation-defined, and there are often implementation-
defined ways to modify it or disable it altogether (i.e., to *pack* the `struct`
instead of *padding* it), usually with stuff like `__attribute__`s or `#pragma`
directives for the preprocessor.

Wait weren't we gonna talk about bit manipulation? Why are we talking about
`struct`s? Well, C does have a workaround to make bit manipulation a little
easier by avoiding that slightly-annoying bit arithmetic you have to do to set
or clear flags in an `int` or `unsigned int`: it's called a *bit field*, and it
takes advantage of `struct` padding.

You can specify the number of bits that a field of a `struct` should occupy by
adding a colon and a size after the field name:
```c
struct bitfield_example {
    unsigned char a : 1;
    unsigned char b : 7;
};
```
This way, you can set the single-bit flag `a` with simple variable assignments
like `var.a = 1`, and the compiler will figure out any necessary magic similar
to structure padding to make that happen. Awesome, right? So why haven't we been
using it all the time instead of all that opaque bit arithmetic with arcane
operators like `<<`, `>>`, `|`, and `&`?

Well, there are some big downsides to bit fields. First, the C standard sets
some strict rules on their use to make sure that compilers can figure out how to
handle them. Bit fields are only allowed inside of structures. You're not
allowed to create arrays of bit fields or pointers to bit fields. Functions
aren't allowed to return a bit field. You're not allowed to get the address of a
bit field with the `&` operator. You can only operate on a single bit field at a
time in any statement; that means you can't set one bit field to equal another,
and you can't compare the values of two bit fields.

Second, they're *extremely* implementation-defined. Each implementation (read:
compiler + architecture combo) determines what data types and sizes are allowed
to be used in bit fields. The data types you *can* use might have different
signedness rules from the usual ones for signed and unsigned types. How they're
laid out, ordered, and padded in memory can differ. In short: the low-level
details are a total black box that you can probably only figure out by reading
*deep* into the compiler's specifications.

Now imagine trying to do something that requires specific protocols like sending
data over a network, and you come across a bit field. Lolwut. Who knows what
you'd have to do. Bit fields make it impossible to port your code.

BUT! Bit arithmetic is annoying, so let's use bit fields anyway!

Okay, so back to `struct gatedesc`. IDT entries have to contain a 16-bit code
segment selector (`%cs`), 16 low bits and 16 high bits for an offset in that
segment, the number of arguments for the handler function, a type, a system/
application flag, a descriptor privilege level (0 for kernel, 3 for user), and a
"present" flag. And x86 is very particular about how it's all laid out, so we
have to set up `struct gatedesc` in the exact right order.
```c
struct gatedesc {
    uint off_15_0 : 16;
    uint cs : 16;
    uint args : 5;
    uint rsv1 : 3;
    uint type : 4;
    uint s : 1;
    uint dpl : 2;
    uint p : 1;
    uint off_31_16 : 16;
};
```

Well, okay, that's it for now.

### tvinit

This function loads all the assembly trap handler functions in `vectors` into
the IDT. The `SETGATE()` macro in
[mmu.h](https://github.com/mit-pdos/xv6-public/blob/master/mmu.h) will organize
each entry correctly. We said before that the IDT needs a code segment selector,
an instruction pointer (from `vectors`), and a privilege level (0 for kernel
mode), so we'll stick those in.
```c
void tvinit(void)
{
    for (int i = 0; i < 256; i++) {
        SETGATE(idt[i], 0, SEG_KCODE << 3, vectors[i], 0);
    }
    // ...
}
```

We're basically done now, but there's one last hiccup: user code needs to be
able to generate system calls, but we just set all the privilege levels so only
the kernel and processor can generate traps. So we'll fix the entry for system
calls as a special case.
```c
void tvinit(void)
{
    // ...
    SETGATE(idt[T_SYSCALL], 1, SEG_KCODE << 3, vectors[T_SYSCALL], DPL_USER);
    // ...
}
```

Oh and while we're at it, let's just go ahead and initialize the lock for the
tick counter.
```c
void tvinit(void)
{
    // ...
    initlock(&tickslock, "time");
}
```

### idtinit

The last function stored all the trap vectors in the IDT, so now we need to tell
the processor where to find the IDT. There's a special assembly instruction for
that in x86 called `lidt`.
```c
void idtinit(void)
{
    lidt(idt, sizeof(idt));
}
```

### trap

This last function is the one that gets called by the assembly code in `alltraps`;
it's responsible for figuring out what to do based on the trap number we pushed
on the stack before. Heads up: it's gonna do that by calling a bunch of other
functions, many of which we haven't seen yet. I'll just give a quick summary
when we come across them, and we'll get to them later on.

The only argument is a pointer to a `struct trapframe`. Wait, hang on. Up above
in the assembly code, the argument we pushed on the stack was `%esp`, the stack
pointer, not a pointer to any `struct trapframe`. What's up with that? Did we
pass the wrong kind of argument in?

Let's check out the definition for `struct trapframe`, found in
[x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h). It's got a
bunch of fields, starting off with the general purpose registers (those are the
fields from `%edi` to `%eax`). Then it has four segment registers (fields `%gs`
through `%ds`), plus some unused padding bits in between them to round the 16-
bit segment registers up to 32 bits. The next two fields are a trap number and
an error code.

All that should sound familiar. Take another look at
[trapasm.S](https://github.com/mit-pdos/xv6-public/blob/master/trapasm.S): so
far, those are the exact same things we pushed on the stack! The other fields
are what the processor pushed on the stack before calling the handler function
in the IDT. So basically, we're never gonna construct a `struct trapframe` in C
code; we already constructed it manually in assembly. It just describes
everything that's already on the stack by the time this `trap()` function gets
called. In that sense, the `%esp` we pushed as an argument really *is* a pointer
to a `struct trapframe`. It's a clever way to read values off the stack.

So we said we're gonna check the trap number and decide which kernel function to
call based on that, right? Let's start by checking if the trap number indicates
this is a system call (trap number 64, or `T_SYSCALL`).
```c
void trap(struct trapframe *tf)
{
    if (tf->trapno == T_SYSCALL) {
        // ...
    }
    // ...
}
```
Well how should we handle system calls? xv6 will have several, and we don't even
know what they all are yet. So let's procrastinate again and just call some
other function `syscall()` to handle the work of figuring out which system call
to execute. Now we'll store the pointer to the `struct trapframe` in that
process's `struct proc`, obtained with a call to `myproc()`. Also, processes
need to be killed once they're done, or if they cause an exception; that happens
by setting a `killed` flag in the `struct proc`. So we'll check for that before
and after carrying out the system call and close the process out with `exit()`
if it's due to be killed.
```c
void trap(struct trapframe *tf)
{
    if (tf->trapno == T_SYSCALL) {
        if (myproc()->killed) {
            exit();
        }
        myproc()->tf = tf;
        syscall();
        if (myproc()->killed) {
            exit();
        }
        return;
    }
    // ...
}
```

Okay, now we have all the other trap numbers to think about. We could do them
with a ton of `if` statements, but that would be a pain; we'll use a `switch`
statement instead. If you haven't seen `switch` statements, they replace big
`if-else` blocks with cases instead. The cases can only be indexed by integers,
and you have to stick a `break` statement at the end or else you'll fall through
to the next case and execute the code found there as well. (To be honest, I
don't see a reason why the system call case wasn't just included in this same
switch statement; if you see a reason for that, let me know.)
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // cases go here
    }
    // ...
}
```

First up is the trap number for timer interrupts; the main function of timer
interrupts is to schedule a new process, but that will come further down in this
function. For now, we'll just increment the `ticks` counter then call `wakeup()`,
which checks if any processes went to sleep until the next tick; it'll switch to
running any process it finds. There's one detail to deal with here: the system
may have multiple processors, each with their own timer and interrupts. We want
to use the `ticks` counter as a rough timer, but we don't know whether all the
CPU timers will be synchronized, so we'll only update `ticks` using the first
CPU to avoid those issues.

If you read the post on interrupt controllers then you'll be familiar with
`lapiceoi()`; if you didn't (or you forgot), it just tells the local interrupt
controller that we've read and acknowledged the current interrupt so it can
clear it and get ready for more interrupts.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        case T_IRQ0 + IRQ_TIMER:
            if (cpuid() == 0) {
                acquire(&tickslock);
                ticks++;
                wakeup(&ticks);
                release(&tickslock);
            }
            lapiceoi();
            break;
        // ...
    }
    // ...
}
```

Later on, we'll see some interrupt handler functions for various devices:
`ideintr()` handles disk interrupts, `kbdintr()` for key presses and releases,
and `uartintr()` for serial port data. We'll direct the corresponding interrupts
to those functions, then acknowledge and clear them with `lapiceoi()`. Also,
devices occasionally generate spurious interrupts due to hardware malfunctions;
we'll either ignore them (if they're coming from the Bochs emulator) or print a
message about it to the console.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // ...
        case T_IRQ0 + IRQ_IDE:      // disk interrupt
            ideintr();
            lapiceoi();
            break;
        case T_IRQ0 + IRQ_IDE + 1:  // spurious Bochs disk interrupt
            break;
        case T_IRQ0 + IRQ_KBD:      // keyboard interrupt
            kbdintr();
            lapiceoi();
            break;
        case T_IRQ + 7:             // spurious interrupt-no break, FALL THROUGH
        case T_IRQ + IRQ_SPURIOUS:  // spurious interrupt
            cprintf("cpu%d: spurious interrupt at %x:%x\n",
                    cpuid(), tf->cs, tf->eip);
            lapiceoi();
            break;
        // ...
    }
    // ...
}
```

Okay, so now we've dealt with system calls and hardware interrupts, so any other
trap must be a software exception. `switch` statements allow a catch-all case
with `default`, so we'll use that to catch the rest of the trap numbers. Now,
this may have come from a kernel error or a misbehaving user process. We can
check with `myproc()`, which returns a null pointer if we were running kernel
code or a pointer to a `struct proc` if we were in user space, or by checking
the current privilege level in the code segment selector. Depending on the
source, we'll print out an appropriate error message and either panic (if in the
kernel) or mark the process so it gets killed soon.
```c
void trap(struct trapframe *tf)
{
    // ...
    switch (tf->trapno) {
        // ...
        default:
            if (myproc() == 0 || (tf->cs & 3) == 0) {
                // Kernel code exception
                cprintf("unexpected trap %d from cpu %d eip %x (cr2=0x%x)\n",
                        tf->trapno, cpuid(), tf->eip, rcr2());
                panic("trap");
            }
            // User process exception
            cprintf("pid %d %s: trap %d err %d on cpu %d "
                    "eip 0x%x addr 0x%x--kill proc\n",
                    myproc()->pid, myproc()->name, tf->trapno,
                    tf->err, cpuid(), tf->eip, rcr2());
            myproc()->killed = 1;
    }
    // ...
}
```
The reason we don't kill it immediately is because the process might be executing
some kernel code right now; for example, system calls allow other interrupts and
exceptions to occur while they're being handled. Killing it now might corrupt
whatever it's doing. So instead we just give it the kiss of death for now and
come back to finish the job later.

So next up, we'll check if this trap was generated by a user process that's due
to be killed, and that process is running in ring 3. If so, we finally do
the deed with `exit()`; otherwise if it's running in ring 0, it'll live for now
and get killed the next time it generates a trap instead.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->killed && (tf->cs & 3) == DPL_USER) {
        exit();
    }
    // ...
}
```

Up above, the only thing a timer interrupt did was increment `ticks`. But we
know a really important function of timer interrupts is to force a process to
let go of the CPU and let someone else run. It's time to do that. We'll check if
the process's state is `RUNNING` and the trap was a timer interrupt; if so, we
call `yield()` to let another process get scheduled on this CPU.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->state == RUNNING &&
            tf->trapno == T_IRQ0 + IRQ_TIMER) {
        yield();
    }
    // ...
}
```

Now we have one last check: a process that yielded, then got picked up again
later might have been marked as killed in the meantime, so if it was, we need to
finish it off now. So we do the exact same check as above again, and then we're
done.
```c
void trap(struct trapframe *tf)
{
    // ...
    if (myproc() && myproc()->killed && (tf->cs & 3) == DPL_USER) {
        exit();
    }
}
```
Note that this function will return into `trapret` in the assembly code, which
will then send it back to user mode.

## Summary

Let's take a moment to assess how much of xv6 we've already covered. Remember,
the xv6 kernel has four main functions: (1) finishing the boot process that the
boot loader started, (2) virtualizing resources in order to isolate processes
from each other, (3) scheduling processes to run, and (4) interfacing
between user processes and hardware devices. Let's take that as a checklist and
go through those items now.

We've already seen some of the initialization routines that get run on boot in
`main()`; most of the code there sets up virtual memory and all the hardware
devices. We still have a few more devices to talk about: the keyboard, serial
port, console, and disk; each of those has its own boot function that we'll need
to go over in order to wrap up point (1).

On the other hand, we're already done with (2) and (3): we spent a lot of time
going over virtual memory and paging, and the last post on scheduling showed us
how xv6 virtualizes the CPU as well as it runs processes.

The code we saw in this post was our introduction to point (4). Traps are the
primary mechanism for user processes to communicate with the hardware; the
kernel coordinates that communication by setting up trap handler functions. The
code we've seen here basically acts like an usher, directing traps to the
right trap handler function depending on its type.

When a trap occurs (x86 instruction `int`), the processor will stop executing
code, find the IDT, and looks up the entry for that trap number. The script that
xv6 uses to generate the IDT entries just makes them all point to the same
function `alltraps()`, which saves all the process's registers, switches into
kernel mode, and calls `trap()`. Then that function uses the trap number to
figure out how the kernel wants it to respond to this particular trap. So any
hardware interrupt, software exception, or user system call will get funneled
into the functions here before getting dispatched to some other appropriate
kernel code that will know what to do with it.

We haven't finished point (4) yet, though: we have to actually see what each of
those trap handler functions does. But we did see some of them: for example, we
saw that a software exception either kills the process that caused it or panics
if it occurred in kernel code. That already takes care of one of the three types
of traps, so we're left with hardware interrupts and system calls. All the
system calls got redirected to a `syscall()` function which we haven't seen yet.

We have seen how some of the hardware interrupts are dealt with: a timer
interrupt increments a `ticks` counter (if it's on CPU 0), then calls `yield()`
to force a process to give up the CPU until the next scheduling round. Spurious
interrupts either get ignored or print a message to the console. But we've
procrastinated some of the others: disk interrupts call an `ideintr()` function
to handle them, keyboard interrupts call `kdbintr()`, and serial port interrupts
call `uartintr()`, none of which we've gone over.

So in order to wrap up the xv6 kernel, we still have to understand how system
calls are routed in general, as well as how devices are initialized at boot and
how the kernel responds to specific system calls that require use of those
devices. The general system call routing mechanism is up next.
