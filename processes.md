# Processes

It's time to turn our attention to processes in xv6!
[proc.c](https://github.com/mit-pdos/xv6-public/blob/master/proc.c) is another
huge file, so I'm gonna split it up into a few posts. This one will focus on the
basic functions we'll need in order to create new processes; later posts will
go over scheduling and system calls.

## proc.h

I haven't spent much time on the header files in xv6, but
[proc.h](https://github.com/mit-pdos/xv6-public/blob/master/proc.h) defines some
important structures we're gonna be using often, so let's just get those out of
the way first.

Let's start off with the definition for `struct context`. The processor will
have to switch between different processes during interrupts, system calls,
exceptions, etc.; these *context switches* will require saving the contents of
some of the CPU registers so that it can reload them when it switches back and
resume execution where it left off. It'll save the process's context by pushing
those register contents on the stack; that way the stack pointer is effectively
a pointer to the context. So the fields of a `struct context` will just list all
the registers that were saved on the stack.

Now, which registers do we need to save? Let's look at the full list on the
[OSDev Wiki](https://wiki.osdev.org/CPU_Registers_x86). We've got some general-
purpose registers, the instruction pointer register `%eip`, segment registers,
a flags register, control registers, and the GDT and IDT registers (x86 doesn't
use the debug, test, or LDT registers).

The flags register, control registers, and GDT/IDT registers shouldn't change
between processes, so we don't need to save those. What about the segment
registers like `%cs`? Back when we set up segmentation, we made the segments be
identity maps that would always stay the same for all processes. There are
separate segments for user mode and kernel mode, but context switches will
always occur in kernel mode, so the segment registers shouldn't change, and we
don't need to save them either.

We should definitely save the program counter (AKA instruction pointer `%eip`),
since that will point to the place in the code where we should resume execution.

The only ones left now are the general-purpose registers: the stack base pointer
`%ebp` and stack pointer `%esp`, along with `%eax`, `%ebx`, `%ecx`, `%edx`,
`%esi`, and `%edi`. We said above that the stack pointer `%esp` would tell us
where to find the context, so that must mean we'll already have it through some
other means in order to find the rest of the context, so we don't need to save
it again (we'll see how we end up getting it later on). But we do need to save
`%ebp`.

There's an x86 convention that the caller of a function has to save `%eax`,
`%ecx`, and `%edx`, so those are already taken care of. So we'll just save the
others: `%edi`, `%esi`, and `%ebx`.

We end up with this list of saved registers as the fields for `struct context`:
```c
// ...
struct context {
    uint edi;
    uint esi;
    uint ebx;
    uint ebp;
    uip eip;
};
// ...
```

Next up: we might end up with a bunch of processes, some of which are currently
running while others aren't. Let's set up some labels to note that. We'll
definitely need a `RUNNING` label; we'll also use one called `RUNNABLE` for
processes that are ready to be run the next time there's a free CPU. We also
need a label for processes that are blocked waiting for something else to happen
(e.g., I/O); xv6 calls this `SLEEPING`. Processes that don't exist yet will be
called `UNUSED`.

There are two special moments in a process's lifecycle that we should be careful
with: birth and death. When we create a new process, we'll have to do a bunch of
setup before it's `RUNNABLE`; killing a process requires clean-up before it goes
back to `UNUSED`. We'll call those `EMBRYO` and `ZOMBIE`, respectively.

We could use bit flags for these states or just regular integers, except then
we'd have to do annoying bit arithmetic or keep track of which number represents
which state. And yes, we could use a bunch of `#define` directives for the
preprocessor for that, but there's a better way to do it. C lets us create data
types for labels using `enum`s. These don't have fields like `struct`s do;
they're basically just a mapping between integers and what the labels those
integers represent. So it's pretty similar to using a bunch of `#define`
directives, except that they're all defined neatly in a single place, so it
helps us remember they're all representing the same idea. So we'll use an `enum`
like this:
```c
// ...
enum procstate { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
// ...
```

Now it's time to look at how we'll represent processes themselves together with
their metadata. Let's see... what kind of unique data does each process have?
We just talked about `struct context`s and `enum procstate`s; each process will
have both of those.

We also talked about virtual memory for processes in a previous post, so it
should also have its own page directory and stack for the kernel to use, plus a
way to track the size of its virtual address space. We said then that processes
are created using `fork()`, so let's add a field to point to the parent process.

We'll need a way for the kernel to refer to a process, so let's give it a unique
process ID. That's not super helpful when it comes to debugging, so let's also
add a name for it as a string.

The rest of the fields are for aspects we haven't seen yet but will talk about
soon: a *trap frame* for interrupts and system calls, a boolean to track whether
a process should be killed soon, a *channel* to be able to wake up a sleeping
process, an array of open files, and a current working directory.
```c
// ...
struct proc {
    uint sz;                    // size (in bytes) of virtual address space
    pde_t *pgdir;               // page directory
    char *kstack;               // kernel stack for this process
    enum procstate state;       // process state
    int pid;                    // process ID
    struct proc *parent;        // parent process
    struct trapframe *tf;       // trap frame for current system call
    struct context *context;    // saved register contents for context switches
    void *chan;                 // channel that process is sleeping on, if any
    int killed;                 // boolean: should process be killed soon?
    struct file *ofile[NOFILE]; // array of open files
    struct inode *cwd;          // current working directory
    char name[16];              // process name
}
// ...
```

Okay, next we'll add another structure for metadata representing each CPU.

If you read the previous post, then you know each CPU has its own local
interrupt controller with a unique ID, so we'll write that down. The post about
process paging talked about the TSS, so we'll need one of those per CPU, plus a
GDT too.

At any point in time, a processor will be running one of: its own initialization
routine (only once while the kernel is setting up), a user process (or any
interrupts or system calls that come up), or a scheduler routine to run the next
process. So let's add a pointer to a `struct proc`, which will be null if it's
not running a process; a boolean `started` will be false until the CPU finishes
its own set-up. The scheduler isn't itself a process; it uses the `kpgdir` page
directory and has its own context, so we'll store that context in a field here.

Finally: remember how the spin-lock post talked about nested calls to `pushcli()`
and `popcli()` tracking whether interrupts were enabled before the first call to
`pushcli()`, and only enabling interrupts after the last call to `popcli()` if
they were enabled before? Those were tracked with per-CPU fields `ncli` and
`intena`, so we need those too.
```c
struct cpu {
    uchar apicid;               // ID of this CPU's local interrupt controller
    struct context *scheduler;  // scheduler's context
    struct taskstate ts;        // task state segment
    struct segdesc gdt[NSEGS];  // global descriptor table
    volatile uint started;      // boolean: has this CPU been initialized yet?
    int ncli;                   // depth of pushcli() nesting
    int intena;                 // were interrupts enabled before pushcli()?
    struct proc *proc;          // currently running process
};
// ...
```

Last but not least, we'll add declarations for the global array of CPUs and the
number of CPUs actually present on this machine; these were defined in
[mp.c](https://github.com/mit-pdos/xv6-public/blob/master/mp.c)

Okay, on to the functions now!

## proc.c

xv6 uses a global process table with an array of processes to store all the
`struct proc`s in; this means we'll never be able to create more processes than
the number of entries in the array, `NPROC`, defined in
[param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 64.
We'll need a lock too to prevent data races while accessing the process table.
The process table's definition does that thing again where you simultaneously
define a `struct type` and define a variable using that type in a single
statement.
```c
struct {
    struct spinlock lock;
    struct proc proc[NPROC];
} ptable;
// ...
```

Then we define a global static variable to point to the first process that gets
run on xv6, so that other files can set it up.
```c
// ...
static struct proc *initproc;
// ...
```

Finally, we're gonna need to assign unique process IDs, so we'll use a global
counter to know which one we should use next.
```c
// ...
int nextpid = 1;
// ...
```

### pinit

This function only does one thing: initializes the lock in the process table.
```c
void pinit(void)
{
    initlock(&ptable.lock, "ptable");
}
```

### mycpu

This function will return a pointer to the `struct cpu` for the current CPU.
There's a potential concurrency bug with this function: if it gets interrupted
before it returns, then it might get rescheduled on a different CPU, and end up
returning an incorrect `struct cpu`. So we need to make sure that interrupts are
disabled when we call it. Normally we'd do that with `pushcli()` and `popcli()`,
but those functions actually call this one, so we'd get an infinite recursion.
So instead we're just gonna have to remember to disable interrupts *before*
calling this function.

If you're reading this because you're gonna do some xv6 kernel hacking for an
OSTEP project or something, you should read that as "DANGER DANGER DANGER!". If
your code calls this function, or calls any other functions that in turn call
this one, you *have* to make sure you've disabled interrupts first.

Concurrency bugs are a nightmare because they're not deterministic: for example,
if you forget to disable interrupts before calling this function, it might work
just fine most of the time until the one unlucky moment when it gets interrupted
and rescheduled on a different CPU. So let's make this easier to debug by
starting off with a check that interrupts are disabled and panic if they're not.
We can check whether the interrupt flag `FL_IF` is set in the `eflags` register.
```c
struct cpu *mycpu(void)
{
    if (readeflags() & FL_IF) {
        panic("mycpu called with interrupts enabled\n");
    }
    // ...
}
```

Okay so how do we figure out which CPU we're on? Well, the previous post talked
about interrupt controllers; each CPU has a local interrupt controller with a
unique ID which we can get with `lapicid()`. Once we have that, we can iterate
over the CPU array `cpus` until we find an entry with a matching `apicid`; we'll
just panic if none of them match.
```c
struct cpu *mycpu(void)
{
    // ...
    int apicid = lapicid();

    for (int i = 0; i < ncpu; ++i) {
        if (cpus[i].apidid == apicid) {
            return &cpus[i];
        }
    }
    panic("unknown apicid\n");
}
```

### cpuid

Those local interrupt controller IDs aren't guaranteed to start from 0, so we'll
need another way to identify CPUs. We can just use its entry number in the
global `cpus` array for that; `cpus` is an array of `struct cpu`s, which in C
means it's really a pointer to the entry with index 0. `mycpu()` returns a
pointer to the entry for the current CPU, so we can just subtract those pointers
to get the index.
```c
int cpuid(void)
{
    return mycpu() - cpus;
}
```

### myproc

This function returns a pointer to the `struct proc` running on this CPU. We're
gonna call `mycpu()` here, so we'll be good and remember to dsable interrupts
first with `pushcli()` and reenable them at the end with `popcli()`. Then we'll
get the current process from the `struct cpu`'s field.
```c
struct proc *myproc(void)
{
    pushcli();

    struct cpu *c = mycpu();
    struct proc *p = c->proc;

    popcli();
    return p;
}
```

### allocproc

Okay, we're finally at the code to create a new process! Whew, it's been a long
journey.

This is a `static` function, which means it can only be called by functions
defined in this same file. Creating a new process will require modifying the
process table, so we need to grab the lock so that other threads can't mess with
it while we're using it.
```c
static struct proc *allocproc(void)
{
    acquire(&ptable.lock);
    // ...
}
```

Now we need to look through the table and find a slot that's `UNUSED`; if we
find on, then great, we'll assign that slot to the new process after the `found`
label below. But if none of them are free, we'll have to return a null pointer
to indicate that. You know what that means, right? Yup, we're gonna have to add
null checks every time we call this function! Wooooo!
```c
static struct proc *allocproc(void)
{
    // ...
    struct proc *p;

    // Look through process table looking for an UNUSED slot
    for (p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == UNUSED) {
            goto found;
        }
    }
    // If none is found, return null pointer
    release(&ptable.lock);
    return 0;

    // ...
}
```
Check out that for loop too: `p` is a pointer to a `struct proc` that starts off
pointing to `ptable.proc`; that means it points to the entry and index 0. Then
it gets incremented by 1 each iteration; since it's a `struct proc`, the pointer
arithmetic will work out so that it points to the next entry in the process
table.

Okay now let's check out the `found` label and see what happens if we did find
an unused slot. First we set its state to `EMBRYO` (instead of `RUNNABLE`, since
we're not done setting it up) and give it a PID. That state means it's neither
`UNUSED` nor `RUNNABLE`, so we can be confident that any other threads wouldn't
try messing with it right now; they can't allocate the slot to another process,
and they can't try to run it yet. So we can stop hogging the process table now
and let other threads take a turn.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    p->state = EMBRYO;
    p->pid = nextpid++;

    release(&ptable.lock);

    // ...
}
```

Now we need to allocate a page for this process's kernel thread to use as a
stack. Remember, `kalloc()` can return null, so we need a null check here.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    if ((p->kstack = kalloc()) == 0) {
        p->state = UNUSED;
        return 0;
    }
    // ...
}
```

Now, we're not gonna set up its page directory yet; that'll happen in `fork()`,
which we'll see later on. But we do need to set up the process so that it'll
start executing code somewhere. It needs to start off in kernel mode, then it'll
context-switch back into user mode and start running its code.

We haven't looked at the mechanics of context switches yet, so I'll spoil it a
little now (I know, I'm sorry). When a process is already running, it can send a
system call to ask for the kernel's attention to do whatever it needs, like a
baby crying until it gets fed or changed or burped or whatever. Then it'll
switch into kernel mode to run the system call, then switch back to where it
left off and pick up from there.

Well, xv6 is all about simplicity, right? And what's more simple and elegant
than treating a special case (creating a new process and starting it off running
some code) the same as the general case (returning from a system call)? So xv6
will set up every new process to start off by "returning" from a (non-existent)
system call. That way the context switch code can be reused for new processes
too.

New processes are created via `fork()`, so we'll return into a function called
`forkret()`. Then that has to return into the function `trapret()`, which
closes out a *trap* (interrupt, system call, or exception) by restoring saved
registers and switching into user mode. We'll get to `forkret()` and `trapret()`
soon.

But first, the challenge: how do we "return" into a function that never called
us in the first place? We talked about function calls in x86 in the post on
spin-locks with the `getcallerpcs()` function, so make sure to read that now if
you need a refresher.

To summarize: when a function `f()` calls another function `g()`, it pushes the
arguments of `g()` on the top of its stack. Then it pushes a return address to
know where it should continue running the code of `f()` after `g()` returns;
that's just the `%eip` register. Then it pushes the base address of the stack
for `f()`, i.e. the current `%ebp` register. That's where `g()`'s stack will
start off.

When the scheduler first runs the new process, it'll check its context via
`p->context` to get its register contents, including the instruction pointer
`%eip`. So if we want it to start executing the code in `forkret()`, the `eip`
field of its context should point to the beginning of `forkret()`. Then we can
trick it into thinking that the previous caller was `trapret()` by setting up
arguments and a return address in its stack.

Let's start off by getting a pointer to the bottom of the stack. We had just
allocated a new stack page at `p->kstack`, but the stack grows from high to low
addresses, so the base of the stack is really at `p->kstack + KSTACKSIZE`. We'll
make it a `char *` so we can move around one byte at a time using pointer
arithmetic.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    char *sp = p->stack + KSTACKSIZE;
    // ...
}
```

Now we should push any arguments for `trapret()` on the stack; it takes a
`struct trapframe` (which we'll go over later), so we'll leave some room for it
and make the process point to it with `p->tf`.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= sizeof(*p->tf);
    p->tf = (struct trapframe *) sp;
    // ...
}
```

Then we add a "return address" to the beginning of `trapret()` after that.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= 4;
    *((uint *) sp) = (uint) trapret;
    // ...
}
```

The last thing we need is to save some space for the process's context on the
stack and point `p->context` to it. Then we'll zero it all out, except for the
`eip` field, which will point to the beginning of `forkret()`. And that's it!
We just return the pointer to the process now.
```c
static struct proc *allocproc(void)
{
    // ...
found:
    // ...
    sp -= sizeof(*p->context);
    p->context = (struct context *) sp;

    memset(p->context, 0, sizeof(*p->context));
    p->context->eip = (uint) forkret;

    return p;
}
```

We can create new processes now!

### growproc

What about growing or shrinking the size of a process's address space? We
already did most of the hard work with `allocuvm()` and `deallocuvm()` from the
post on process paging, so let's take a beat to thank past us for that.

Okay, so first we have to get the current process's size.
```c
int growproc(int n)
{
    struct proc *curproc = myproc();
    uint sz = curproc->sz;
    // ...
}
```

Depending on the size of `n`, we'll either grow the process or shrink it by `n`
bytes. Both `allocuvm()` and `deallocuvm()` can fail and return zero, so let's
add some checks for those and return -1 if they fail.
```c
int growproc(int n)
{
    // ...
    if (n > 0) {
        if ((sz = allocuvm(curproc->pgdir, sz, sz + n)) == 0) {
            return -1;
        }
    } else if (n < 0) {
        if ((sz = deallocuvm(curproc->pgdir, sz, sz + n)) == 0) {
            return -1;
        }
    }

    curproc->sz = sz;
    // ...
}
```

Finally, we need to tell the hardware that there's a new page directory in town
with a different size than the old one, so we'll use `switchuvm()` to update
the page directory and TSS stored by the hardware to reflect the changes. Then
we return 0 to indicate everything went okay.
```c
int growproc(int n)
{
    // ...
    switchuvm(curproc);
    return 0;
}
```

### procdump

This function is for debugging purposes: it'll print a complete listing of any
processes in the process table. Quick spoiler: the keyboard interrupt handler
function will set things up so that pressing `^P` runs this function. Go ahead,
load up xv6 and try it out!

We want to print out the state for each process, but the states in `enum
procstate` are just integers, which isn't very debug-friendly. So let's map them
all to strings first with a static array of strings.
```c
void procdump(void)
{
    static char *states[] = {
        [UNUSED]    "unused",
        [EMBRYO]    "embryo",
        [SLEEPING]  "sleep ",
        [RUNNABLE]  "runble",
        [RUNNING]   "run   ",
        [ZOMBIE]    "sombie",
    };
    // ...
}
```
This array notation might be a little unusual if you haven't seen it before: C
lets you initialize arrays by specifying the value of each entry. If you leave
any entries out, then they'll get initialized to zero. You can even write the
entries out of order by adding their index before them in square brackets. So
`{ [1] 5, [0] 2 }` is the same thing as `{2, 5}`. The `enum` turns the states
into integers, so they work as indices here.

Now we'll just iterate over the process table to get all the processes, skipping
over any `UNUSED` ones.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == UNUSED) {
            continue;
        }
        // ...
    }
}
```

Next we'll get the process's state (or just use `"???"` if something went wrong
and the state isn't recognized).
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        char *state;
        if (p->state >= 0 && p->state < NELEM(states) && states[p->state]) {
            state = states[p->state];
        } else {
            state = "???";
        }
        // ...
    }
}
```

Then we can print out its PID, state, and name to the console.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        cprintf("%d %s %s", p->pid, state, p->name);
        // ...
    }
}
```

Finally, we'll see later on that the `sleep()` and `wakeup()` system calls
involve some lock trickery, so sleeping processes could be a common cause of
concurrency issues like deadlocks. So if a process is sleeping, we'll print out
its call stack using the `getcallerpcs()` function.
```c
void procdump(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        // ...
        if (p->state == SLEEPING) {
            uint pc[10];
            getcallerpcs((uint *) p->context->ebp + 2, pc);

            for (int i = 0; i < 10 && pc[i] != 0; i++) {
                cprintf(" %p", pc[i]);
            }
        }
        cprintf("\n");
    }
}
```

## Summary

Whew, we're making good progress. The most important part of this code was how
xv6 creates new processes and sets them up to start running: basically, it uses
some stack and function call trickery to make the scheduler start running a new
process with the code in `forkret()`, then `trapret()`, before switching context
into user mode.

We haven't talked about those two functions yet; we'll hold off on that until we
do traps and system calls. Next up is scheduling processes!
