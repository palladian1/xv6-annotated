# Scheduling

We've done a lot of talking about context switching and scheduling, but we've
procrastinated looking at the code for those. It's time to fix that.

There are all kinds of advanced schedulers out there, but as we've said before,
the name of the game in xv6 is simplicity, so xv6 just uses a round-robin
scheduling algorithm in which in loops through the exisitng processes in order.
Each timer interrupt will force the current process to yield the processor and
perform a context switch back into the scheduler so it can run the next
available process.

## swtch.S

The `struct context` we talked about in the last post is gonna be key here, so
let's just look at its fields again:
```c
struct context {
    uint edi;
    uint esi;
    uint ebx;
    uint ebp;
    uint eip;
};
```

The context switch function is `swtch()`; it's gonna need to save and restore
processor registers, so that means it's gonna have to be written in assembly.
But let's pretend it's just a C function for a second and talk about what it's
going to do.

This function will save the contents of the registers on the stack as a `struct
context`, then save that location as the old context. Then it'll load a new
context, switch to the new stack, and restore the registers of the new context.
Its declaration would look like this in C:
```c
void swtch(struct context **old, struct context *new);
```
The first argument is a pointer *to a pointer* to a `struct context`. That
double indirection might be confusing, but there's a method to this madness: C
passes arguments by value, so if we used `struct context *old` and changed `old`
to point to the saved context, it would be lost as soon as we returned from this
function. So instead we have to use this kind of double pointer so we can set
`*old` to point to the saved context. This way `old` will be lost anyway, but
`*old` was changed and will persist beyond this function's return.

Note that, as we've said before, those arguments will be pushed on the stack
before `swtch()` is called. So at the beginning of `swtch()`, the stack pointer
`%esp` points to a return address; the argument `old` is one space (4 bytes)
above that in the stack, and `new` is one space higher than that.

Okay, let's check out the assembly code now. We're gonna start by saving those
arguments into registers. We can't just use any old registers here, or we might
overwrite some of the data we're trying to save. But in the last post, I said
x86 has a convention that the caller has to save the contents of the `%eax`,
`%ecx`, and `%edx` registers, so that means we're free to overwrite them all we
want since they've already been saved.
```asm
.globl swtch
swtch:
    movl    4(%esp), %eax
    movl    8(%esp), %edx
    # ...
```
We haven't seen this number-parenthesis notation in assembly yet, so in case
you're not familiar with x86 assembly, it's just a way to add a number to the
contents of a register, then treating it as a pointer and dereferencing it. So
`4(%esp)` in assembly is the same as `*(esp + 4)` in C. So at this point, `%eax`
holds the `struct context **old` pointer, and `%edx` holds the
`struct context *new` pointer.

Now it's time to save all the fields in a `struct context` on the stack. The
stack grows from high addresses to low ones, but C `structs` expect their fields
to be from low to high, so we'll save them in reverse order. Oh, and hang on --
remember what's at the bottom of the stack right now, after the arguments?
That's right, a return address. That's just a saved `%eip`, so that one's
already done for us! We just need to save the others.
```asm
swtch:
    # ...
    pushl   %ebp
    pushl   %ebx
    pushl   %esi
    pushl   %edi
    # ...
```

Next we have to save a pointer to this old `struct context` into `*old`. Well,
we pushed them on the stack in reverse order, right? So `%esp` already *is*
pointing to it, so that's our pointer; we'll just copy it into `*old` (remember
it's stored in `%eax`, and we dereference it in assembly with parentheses).
```asm
swtch:
    # ...
    movl    %esp, (%eax)
    # ...
```

Now it's time to switch stacks to the `new` context, which we saved in `%edx`.
That context must have been saved by a previous call to `swtch()`, so it also
happens to be a stack pointer as well.
```asm
swtch:
    # ...
    movl    %edx, %esp
    # ...
```

At this point, we're using the stack from `new`, which will already have its
saved context at the top. So we can load the new context by popping it off the
stack in reverse order into the corresponding registers. And again, just like
the `call` instruction had already saved `%eip` on the stack as the return
address, the `ret` (return) instruction will pop it off and restore it into
`%eip` for us.
```asm
swtch:
    # ...
    popl    %edi
    popl    %esi
    popl    %ebx
    popl    %ebp
    ret
```

And that's it! That's a context switch in xv6.

## proc.c

And now, finally, we can look at the scheduling code. Once the kernel is done
setting itself up, initializing all the devices and drivers, etc., the very last
function that `main()` calls is `scheduler()`. Interrupts were disabled in the
boot loader and haven't been enabled yet, so it's also the scheduler's job to
enable them for the first time in xv6.

`scheduler()` never returns; it's an infinite loop that just keeps searching
through the process table for a `RUNNABLE` process, then runs it. So from that
point on, with the exception of interrupts and system calls, the kernel will
only ever do one thing: schedule processes to run.

### scheduler

A CPU that's running the scheduler isn't running its own process. So we'll start
off by setting this CPU's process pointer to null. Note that `mycpu()` requires
interrupts to be disabled before it's called, but that's okay here because
interrupts were disabled in the boot loader and haven't been re-enabled before
the scheduler is called.
```c
void scheduler(void)
{
    struct cpu *c = mycpu();
    c->proc = 0;
    // ...
}
```

The order of the next few steps is tricky, and the authors of xv6 had to be
extremely careful to do them in the right order to avoid concurrency problems.
We need to (1) re-enable interrupts, (2) acquire the process table's lock, and
(3) create an infinite loop to iterate over the process table forever, scheduling
processes along the way. To see why this is nontrivial, let's check out some
different orders (with a `fake_scheduler()` function) and see what problems we
get.

ATTEMPT #1: interrupts -> lock -> loop. Let's try it out.
```c
void fake_scheduler1(void)
{
    // ...
    sti();                  // enable interrupts
    acquire(&ptable.lock);  // acquire lock
    for (;;) {              // infinite scheduling loop
        // ...
    }
}
```
Interrupts have been disabled since the boot loader used `cli`, so when we call
`sti()` here they'll be turned on for the first time in the kernel. At that
point we'll find out if there were any interrupts waiting to be acknowledged,
and possibly jump into some handler function to take care of it. Then when
that's done, we'll come back here and acquire the process table's lock. Acquiring
a lock disables interrupts, remember? So they're disabled again in the infinite
scheduling loop (but not forever; we'll release the lock before switching to a
user process). That sounds okay, right?

Not so fast! There's a hidden problem: suppose we had a situation in which none
of the current processes are `RUNNABLE` -- maybe they're all blocked (or
`SLEEPING`) waiting for I/O or something, which is not unlikely. In that case,
the scheduler would just keep idly looping through the process table until one
of them becomes `RUNNABLE` again. But if interrupts are always disabled in the
loop, then this processor will never find out about, e.g., a disk interrupt
saying it's done reading data which would allow a blocked process to become
`RUNNABLE`. That means the process will never find out the condition it's
waiting for has already happened, which means the scheduler will never find any
`RUNNABLE` processes. It'll just get stuck in an infinite loop, repeatedly and
desperately searching every entry of the process table. So basically, the
system would freeze while the CPU pointlessly spins at top speed.

Okay okay, so that doesn't work. We'll have to periodically re-enable interrupts
before disabling them again. So let's try moving the call to `sti()` inside the
infinite loop so interrupts get re-enabled every once in a while.

ATTEMPT #2: lock -> loop -> interrupts.
```c
void fake_scheduler2(void)
{
    // ...
    acquire(&ptable.lock);  // acquire lock
    for (;;) {              // infinite scheduling loop
        sti();              // temporarily enable interrupts
        // ...
    }
}
```
Problem solved, right? Actually... this one turns out to be just as bad. The
call to `acquire()` disables interrupts, only for `sti()` to enable them again.
There's a reason that locks disable interrupts, remember? If an interrupt occurs
that switches away from `scheduler()`, then it might call a handler function
that needs to access the process table lock, which is already held by
`scheduler()`, so that function would spin forever in a deadlock.

So now we arrive at the correct order: we'll call *both* `sti()` and `acquire()`
inside the loop, in that order. That means we'll also need a call to `release()`
at the end of the loop before we try to `acquire()` again in the next iteration.
We had already said we'd have to release the lock before running a process; now
we'll have to acquire it again before context-switching back into the loop.

ATTEMPT #3 (the right one): loop -> interrupts -> lock. This will give us a
chance to detect any outstanding interrupts in each iteration of the for loop,
but before we've acquired the lock again and thus, before doing so could cause a
deadlock.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        sti();
        acquire(&ptable.lock);
        // ... pick a process and run it ...
        release(&ptable.lock);
    }
}
```

Whew, okay. Basically, we've learned that concurrency bugs can be hard to predict
and can turn seemingly-fine code into impossible-to-diagnose system crashes or
freezes.

Okay, so now let's fill in the part of the loop where the scheduling algorithm
goes. We'll add an inner for loop to iterate over the process table entries
and stop when we find a `RUNNABLE` process.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            if (p->state != RUNNABLE) {
                continue;
            }
            // ... run that process ...
        }
        // ...
    }
}
```

Next, if we found a process, then we need to switch to that process's virtual
address space; that is, we need to start using its page directory.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            c->proc = p;
            switchuvm(p);
            // ...
        }
        // ...
    }
}
```
Now, if we just switched to an arbitrary page directory in the middle of running
other code, we might cause a bunch of problems: all the virtual addresses we're
currently using for variables, functions, instructions, etc. might suddenly
become invalid and point to random other places in memory. But this is where can
see some of the earlier design decisions in xv6 start to pay off: remember how
`setupkvm()` made sure every single process would have the exact same mappings
for the upper half of the address space, starting at `KERNBASE`? That means that
if we're running in kernel mode, we can arbitrarily switch to any process's page
directory and know that all of our mappings will be exactly the same. The user
mappings in the lower half might be different, but the kernel side will never
change. Nice!

Now we can run the process using `swtch()`.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            p->state = RUNNING;
            swtch(&(c->scheduler), p->context);
            // ...
        }
        // ...
    }
}
```
`swtch()` will *not* return here immediately; instead, it'll pick up execution
wherever the process last left off, which will be in kernel mode -- if it
stopped running before, it must have been due to a system call, interrupt, or
exception, which would have been handled in kernel mode before calling the
scheduler.

Note that this process will still be holding the process table lock when it
starts running again. For example, that's the main reason for the existence of
the `forkret()` function we mentioned before. This is another dangerous detail
we'll have to remember, so I'm just gonna go ahead and hope you remember THIS
BIG GIANT GLARING WARNING FLAG RIGHT HERE: if you do any xv6 kernel hacking, and
you want to add a new system call that will let go of the CPU, then your code
*must* release the process table lock at the point at which it starts executing
after switching to it from the scheduler.

This is pretty dangerous; if xv6 were a big project, it would be really easy to
forget that when adding more features later on. But in this case, there's no
easy way to get around it; for example, we can't just release the process table
lock before calling `swtch()` and reacquire it after. The problem becomes
apparent if you think of locks as protecting some invariant; that invariant
might be temporarily violated while you hold the lock, but it should be restored
before the lock is released.

The process table protects invariants related to the process's `p->state` and
`p->context` fields, e.g. that the CPU registers must hold the process's
register values, that a `RUNNABLE` process must be able to be run by any idle
CPU's scheduler, etc. These don't hold true while executing in `swtch()`, so we
need to hold the lock then; otherwise another CPU might decide to run the
process before `swtch()` is done executing.

Now, at some point that process will be done running and will give up the CPU
again. Before it switches back into the scheduler, it has to acquire the process
table lock again. So here's ONE MORE GIANT WARNING for good measure: you should
make sure to do that too if you add your own scheduling-related system call.

Eventually, it'll switch back here with a call with the arguments in reverse,
like `swtch(&(p->context), c->scheduler)`. At the point, execution of the
scheduler will resume right here, so we need to switch back to using the kernel
page directory `kpgdir`.
```c
void scheduler(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            switchkvm();
            c->proc = 0;
        }
        // ...
    }
}
```

After that, the outer for loop just releases the lock before looping to the top
again to temporarily re-enable interrupts, then acquire the lock again and check
for another process to run.

### forkret

Let's take a quick look at one example of where a process might start to execute
after being scheduled. All processes (whether the very first process, or any
others created later through calls to `fork()`) will start running code in
`forkret()`, then return from here into `trapret()`.

Most of the time, this function does just one thing: it releases the process
table lock. However, there are two kernel initialization functions that have to
be run from user mode, so we can't just call them from `main()` and be done with
it. We need a place for a process to call them, and `forkret()` is as good a
place as any. So the very first call to `forkret()` will run these two start-up
functions, and the rest will ignore them.

The two functions are `iinit()` and `initlog()`, which are part of xv6's file
system code; we'll get to them later on. For now, we'll just use a `static int`
as a boolean and set it to false after we've run those functions once on our
first pass through `forkret()`.
```c
void forkret(void)
{
    static int first = 1;

    release(&ptable.lock);

    // Only gets run once, on the first call
    if (first) {
        first = 0;
        iinit(ROOTDEV);
        initlog(ROOTDEV);
    }
    // Returns into `trapret()`
}
```
Any other kernel code that switches into the scheduler (e.g., `sleep()` and
`yield()`) will have a similar lock release right after returning from
the scheduler.

### sched

We saw one example of code that runs after switching *away* from the scheduler,
but what about code that runs before switching *to* the scheduler? Any functions
that need to call into the scheduler can't just call `scheduler()`, since the
scheduler probably left off last time halfway through the loop and should resume
in the same place. So `sched()` handles the task of picking up the scheduler
wherever it last left off.

`sched()` should be called *after* acquiring the process table lock and without
holding any other locks (lest we cause a deadlock somewhere). Also, the process
should not be in the `RUNNING` state anymore since we're about to stop running
it. So we'll start off by checking that those are all true and that interrupts
are disabled.
```c
void sched(void)
{
    struct proc *p = myproc();

    if (!holding(&ptable.lock)) {
        panic("sched ptable.lock");
    }
    if (mycpu()->ncli != 1) {
        panic("sched locks");
    }
    if (p->state == RUNNING) {
        panic("sched running");
    }
    if (readeflags() & FL_IF) {
        panic("sched interruptible");
    }
    // ...
}
```

Next, remember when the `pushcli()` and `popcli()` functions checked whether
interrupts were enabled before turning them off while holding a lock? That's
really a property of this kernel thread, not of this CPU, so we need to save
that now. Then we can call `swtch()` to pick up where the scheduler left off
(the line right after its own call to `swtch()`). This process will resume
executing after that line eventually, at which point we'll restore the data
about whether interrupts were enabled and let it run again.
```c
void sched(void)
{
    // ...

    // Save whether interrupts were enabled before acquiring the lock
    int intena = mycpu()->intena;

    // Perform context switch into the scheduler
    swtch(&p->context, mycpu()->scheduler);
    // Execution will eventually resume here

    // Restore whether interrupts were enabled before
    mycpu()->intena = intena;
}
```

### yield

Okay, let's see an example of how this all comes together now! The `yield()`
function forces a process to give up the CPU for one scheduling round. For
example, this will be used to handle timer interrupts later on. Now that we know
how scheduling works in xv6, `yield()` is easy. We just acquire the process
table lock, set the current process's state to `RUNNABLE` so it can get picked
up again in the next scheduling round, and call `sched()` to switch into the
scheduler. When we eventually return here, we'll just release the lock again.
```c
void yield(void)
{
    acquire(&ptable.lock);

    myproc()->state = RUNNABLE;
    sched();

    release(&ptable.lock);
}
```

## Summary

We've now seen how xv6 handles process scheduling with a super-simple round-
robin algorithm. The `scheduler()` function had plenty of concurrency pitfalls,
but luckily the xv6 authors took care of all the careful coding for us, so we
just get to sit back and admire their work.

We also saw how context switches occur in xv6, so now we can understand how, in
the previous post, `allocproc()` set up a new process with a context that would
result in it starting execution in `forkret()`.

Next up, we'll look at the way xv6 handles interrupts, system calls, and software
exceptions.
