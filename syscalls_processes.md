# System Calls: Processes

In a previous post, I pointed out some of the most important functions a kernel
has to fulfill. System calls take care of two of these: virtualizing resources
via virtual memory and processes, and mediating communication between user-mode
processes and the hardware. We'll wrap up the former now by looking at the
system call functions relating to processes and scheduling.

## proc.c

### fork

Unlike some of the other functions we'll talk about in this post, `fork()` is
used almost exclusively by user code as a system call; the kernel never calls
it. That said, it has an extremely important role: after the first process has
started, it's the only way to create more processes. It does that by copying the
parent process's virtual address space into a new page directory. We haven't
talked about the file system yet, but hopefully you're familiar with file I/O in
Linux, so you know each process has its own list of open files and a current
working directory; `fork()` will clone those as well for the child process.

Let's start off by getting a pointer to the parent process and creating a slot
in the process table for the child process with `allocproc()`. Remember, that
function returns a pointer to the new process's `struct proc`, but it can fail
and return null (e.g., if there is no available slot in the process table, or if
its call to `kalloc()` fails), so we'll need to check for that.
```c
int fork(void)
{
    // Parent process
    struct proc *curproc = myproc();

    // Allocate process table slot for child process
    struct proc *np;
    if ((np = allocproc()) == 0) {
        return -1;
    }
    // ...
}
```
`allocproc()` also sets up the new process's stack so that it'll return into
`forkret()`, then `trapret()`, before context switching into user mode, and sets
the process's state to `EMBRYO`.

Next we need a page directory for the new child process; it should be a copy of
the parent process's page directory. Luckily, we already did the hard work for
this back in the virtual memory posts, so we can just use `copyuvm()` now. That
function can also fail, in which case we'll free the stack that `allocproc()`
created and set the child process's state back to `UNUSED`.
```c
int fork(void)
{
    // ...
    if ((np->pgdir = copyuvm(curproc->pgdir, curproc->sz)) == 0) {
        kfree(np->kstack);
        np->kstack = 0;
        np->state = UNUSED;
        return -1;
    }
    // ...
}
```

Next we'll copy the parent process's size and trap frame; the latter will make
sure the child starts executing after `trapret()` with the same register
contents as the parent. We'll set the child process's parent to, well, its
parent (the current process).
```c
int fork(void)
{
    // ...
    np->sz = curproc->sz;
    np->parent = curproc;
    *np->tf = *curproc->tf;
    // ...
}
```

The two processes will be nearly identical, so we need a way to distiguish them
from user space so that a user program can give different instructions to each.
xv6 follows the Unix convention that `fork()` should return the child process's
PID to the parent and return 0 for the child. The parent's return value is easy;
we'll just literally return the child's PID at the end. But the child didn't
actually call `fork()`, so how can we set a return value that it will see?

Well, the x86 convention is for return values to be passed in the `%eax`
register, right? And that register will be restored from the trap frame before
switching into user mode. So we'll just store the value 0 there.
```c
int fork(void)
{
    // ...
    np->tf->eax = 0;
    // ...
}
```

Next we'll copy all the parent process's open files and its current working
current working directory. The files are stored in a per-process file array
`curproc->ofile` of size `NOFILE`, so we can copy them over with the function
`filedup()` (which we'll see later). The current working directory is in
`curproc->cwd` and can be copied with `idup()`.
```c
int fork(void)
{
    // ...
    for (int i = 0; i < NOFILE; i++) {
        if curproc->ofile[i]) {
            np->ofile[i] = filedup(curproc->ofile[i]);
        }
    }
    np->cwd = idup(curproc->cwd);
    // ...
}
```

Then we'll copy the parent process's name with `safestrcpy()`, defined in
[string.c](https://github.com/mit-pdos/xv6-public/blob/master/string.c). You
might be familiar with the C standard library funtion `strncpy()`; this function
is almost identical, except that unlike `strncpy()` it's guaranteed to nul-
terminate the string it copies. If you haven't seen this kind of thing before,
it's a fairly common practice to write your own safe wrappers for some of the C
standard library functions, especially the ones in `string.h` which are so often
error-prone and dangerous.
```c
int fork(void)
{
    // ...
    safestrcpy(np->name, curproc->name, sizeof(curproc->name));
    // ...
}
```

Finally, we'll set the child process's state to `RUNNABLE` and return its PID
for the parent.
```c
int fork(void)
{
    // ...
    int pid = np->pid;

    acquire(&ptable.lock);
    np->state = RUNNABLE;
    release(&ptable.lock);

    return pid;
}
```

### kill

This is one of the functions that can get called both by the kernel and as a
system call. The kernel will use it to terminate malicious or buggy processes,
and user code can use it as a system call to kill another process too.

We said before that killing a process immediately would present all kinds of
risks (e.g. corrupting any kernel data structures it might be updating, etc.),
so all we're gonna do is give it the ominous mark of death with the `p->killed`
field. Then the code in `trap()` will handle the actual murder the next time the
process passes through there.

The argument is a process ID number, so let's just iterate over the process
table until we find a process with a matching PID; we'll return -1 if we don't
find any.
```c
int kill(int pid)
{
    acquire(&ptable.lock);

    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->pid == pid) {
            // ...
        }
    }

    release(&ptable.lock);
    return -1;
}
```

If we do find a matching process, then we'll set `p->killed`. Also, some of the
calls to `sleep()` will occur inside a while loop that checks if `p->killed` has
been set since the process started sleeping, so let's hasten the process's death
a little by setting its state to `RUNNABLE` so it'll wake up and encounter those
checks faster. There's no risk of screwing up by waking up a process too early,
since each call to `sleep()` should be in a loop that will just put it back to
sleep if it's not ready to wake up yet.
```c
int kill(int pid)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->pid == pid) {
            p->killed = 1;

            if (p->state == SLEEPING) {
                p->state = RUNNABLE;
            }

            release(&ptable.lock);
            return 0;
        }
    }
    // ...
}
```

### sleep

The last post went over the basics of `sleep()` and `wakeup()`; they act as
mechanisms for *sequence coordination* or *conditional synchronization*, which
allows processes to communicate with each other by sleeping while waiting for
conditions to be fulfilled and waking up other processes when those conditions
are satisfied.

Processes can go to sleep on a channel or wake up other processes sleeping on a
channel. In many operating systems, this is achieved via channel queues or even
more complex data structures, but xv6 makes it as simple as possible by simply
using pointers (or equivalently, integers) as channels; the kernel can just use
any convenient address as a pointer for one process to sleep on while other
processes send a wakeup call using the same pointer.

This does mean that multiple processes might be sleeping on the same channel,
either because they are waiting for the same condition before resuming execution
or because two different `sleep()`/`wakeup()` pairs accidentally used the same
channel. The result would be that a process might be woken up before the
condition it's waiting for has been fulfilled. We can solve that problem by
requiring every call to `sleep()` to occur inside a loop that checks the
condition; that way, if a process receives a spurious wakeup call before it
really should have been woken up, the loop will put it right back to sleep
anyway. We saw one example of this in the `sys_sleep()` function, in which the
while loop checked if the right number of ticks had passed.

A common concurrency danger with conditional synchronization in any operating
system is the problem of missed wakeup calls: if the process that's supposed to
send the wakeup call runs *before* the process that's supposed to sleep, it's
possible that the sleeping process will never be woken up again. The problem is
more general than just processes; it applies to devices too.

Imagine this scenario: a process tries to read from the disk; it'll check
whether the data is ready yet and go to sleep (inside a while loop) until it is.
If the disk gets to run first, then the process will just find the data ready
and waiting for it, so it can continue on to use the data. If the process runs
before the disk does, then it'll see the data isn't ready yet and sleep in a
loop until it is; the disk will wake the process up once the data is ready.

But suppose they run at the same time, or in between each other. The process
does its check and finds the data isn't ready, but before it can go to sleep, a
timer interrupt or some other trap goes off and the kernel switches processes.
*Then* the disk finishes reading and starts a disk interrupt that sends a wakeup
call to any sleeping processes, but the process isn't sleeping yet. When the
process starts running again later on, it'll go to sleep -- having already
missed its wakeup call.

The problem is that the process can get interrupted between checking the
condition and going to sleep, right? So why don't we just disable interrupts
there with `pushcli()` and `popcli()`? add a lock there? Ah, but there's another
problem: what if the disk driver is running simultaneously on another CPU?
Disabling interrupts on the process's CPU wouldn't stop the other CPU from
sending the disk's wakeup call too early.

Okay fine, so let's use a lock instead. The process will hold the lock while it
checks the condition and sleeps, and the disk driver will have to acquire the
lock before it can send its wakeup call... Can you see the problem here? If the
process holds the lock while it's sleeping, the disk driver will never be able
to acquire the lock in order to wake it up. That's a deadlock.

HEAD. DESK.

Ugh, okay, fine, you got me. So let's use a lock, but let's have `sleep()`
release it right away, then reacquire it before waking up; that way the lock
will be free while the process is sleeping so the disk driver can acquire it.
Done, right? Everybody's happy?

Nope. Now we're back to the original problem: if the lock gets released inside
`sleep()` before the process is actually sleeping, then the wakeup call might
happen in between those and get missed.

@*#&@#$**&@#%$!!!

So we need a lock. And we can't hold the lock while sleeping, or we'd get a
deadlock. But we also can't release it before sleeping, or we might miss a
wakeup call. So... ???

See, I told you: concurrency is your worst nightmare. Ever since we decided we'd
like our operating systems to do more than run a single basic process at a time,
we introduced all *kinds* of problems we have to reason through. Let's check out
how xv6 actually writes the `sleep()` function and think through it ourselves
and try to understand if it manages to solve this problem.

We'll start by making sure of two things: (1) this CPU is currently running a
process and not the scheduler (which can't ever go to sleep), and (2) the caller
passed in a lock (which can be any arbitrary lock).
```c
void sleep(void *chan, struct spinlock *lk)
{
    struct proc *p = myproc();
    if (p == 0) {
        panic("sleep");
    }
    if (lk == 0) {
        panic("sleep without lk");
    }
    // ...
}
```

Next we need to release the lock and put the process to sleep. That will require
modifying its state, so we should now acquire the lock for the process table.
But if the lock that the process is already holding *is* the process table lock,
then trying to acquire it again would cause a panic, so let's add a check for
that; if we're already holding it then we'll keep using it and we don't need to
release it.
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    if (lk != &ptable.lock) {
        acquire(&ptable.lock);
        release(lk);
    }
    // ...
}
```

Okay, now it's nap time for this process. We just update its channel to `chan`
and its state to `SLEEPING`, then call `sched()` to perform a context switch
into the scheduler so it can run a new process. We *have* to be holding the
process table lock before calling `sched()`, remember?
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    p->chan = chan;
    p->state = SLEEPING;
    sched();
    // ...
}
```

When the process wakes up later on (if indeed it turns out that the code here
works and doesn't miss any wakeup calls), it'll eventually be run by the
scheduler, at which point it will context switch back here. So at that point
we'll reset its channel and reacquire the original lock before returning.
```c
void sleep(void *chan, struct spinlock *lk)
{
    // ...
    p->chan = 0;
    if (lk != &ptable.lock) {
        release(&ptable.lock);
        acquire(lk);
    }
}
```

Okay, well I don't know about you, but I'm still not convinced that this
implementation won't miss any wakeup calls. After all, we release the original
lock before putting the process to sleep, right? We're holding the process table
lock at that point, which at least means that interrupts are disabled, but the
process that will wake this one up might already be running on another CPU and
might send the wakeup signal in between releasing the original lock and
updating this process's channel and state. Hmm... Well, as always, xv6 is
brilliant, so we'll see how this gets solved in the code for `wakeup()`.

But wait! Before we move on, I have a warning for you about using this function
in your own code when you start hacking away at xv6. Remember that when we first
talked about deadlocks, we saw we can cause a deadlock if two processes acquire
two locks in opposite orders? If process 1 tries to acquire lock A, then lock B,
and process 2 simultaneously tries to acquire lock B, then lock A, then the end
result is that process 1 will acquire lock A and process 2 will acquire lock B,
but neither will be able to acquire the other lock since it's already being held.

If you look at the code above, the process that called `sleep()` must have
already been holding a lock `lk`, then `sleep()` acquires `ptable.lock` before
releasing `lk`. You know what that means: there's potential for a deadlock. So
in order to avoid that, you should make sure that *any* lock you pass in to
`sleep()` must *always* get acquired before `ptable.lock`. If any other function
(or chain of function calls) could potentially acquire `ptable.lock` before `lk`,
then you might end up with a deadlock. As always, the xv6 authors have been
extremely careful to make sure that that never happens in the existing code, so
you'll have to do the same thing for any code you add.

### wakeup

This function is short and sweet because it procrastinates all the work it has
to do by pushing it off to a helper function, `wakeup1()`. It just acquires the
process table lock, calls `wakeup1()`, then releases the process table lock. It
has to grab that lock since it's gonna modify the process's state in the process
table.
```c
void wakeup(void *chan)
{
    acquire(&ptable.lock);
    wakeup1(chan);
    release(&ptable.lock);
}
```
xv6 has to use this kind of a wrapper function for the real wakeup function
`wakeup1()` in order to let processes that are already holding the process table
lock send wakeup calls too.

Okay, now before we go look at `wakeup1()`, let's get back to figuring out
whether xv6's implementation of `sleep()` and `wakeup()` can lead to missed
wakeup calls. Take a look at the code in `sleep()` again where the original lock
gets released -- we have to acquire the process table lock *before* we can
release the other lock. So now there are always two locks in play whenever we
use `sleep()` and `wakeup()`.

Let's go back to the example of a process waiting on a disk read. The process
acquires some disk-related lock first, then checks to see if the disk is done
reading; if not, it'll call `sleep()` inside a while loop. If the disk driver
runs now before the process gets to call `sleep()`, that's okay: the disk driver
also has to acquire the same lock before calling `wakeup()`, so the disk would
just end up spinning idly. Eventually, the process runs again and gets to
call `sleep()`; there, it will first acquire the process table lock before
releasing the original disk-related lock.

So what happens if the disk driver's code runs now? Now the disk would be able
to acquire the original lock, so there's nothing stopping it from calling
`wakeup()`. But the very first thing it has to do there is acquire the process
table lock, which the process is already holding, so it just spins idly again!
There's no way the disk driver could ever beat the process to acquiring this
second lock, because the process already held the first (disk-related) lock
before acquiring the second one (the process table lock). Now the process can
finish going to sleep and switch into the scheduler, which will eventually
release the process table lock. So then the disk driver can acquire it, release
the first lock, and finally send its wakeup call.

Moral of the story? There's no way for xv6 to ever have any missed wakeup calls!
The trick was to use two locks, and acquire the second before releasing the
first. But coming up with that solution isn't as easy as saying "oh, just use
two locks!" The solution only works because of the way the process table lock is
already being handled by so many other parts of the kernel code. For example, if
the context switch into the scheduler wasn't guaranteed to release the process
table lock, then the disk driver in the example would never be able to acquire
it after the process goes to sleep, resulting in a deadlock. The solution works
because of all the design decisions in xv6 up to this point.

### wakeup1

Okay, I'll stop fawning over the intricacies of xv6 concurrency management now
so we can look at how wakeup calls actually happen. Remember, this is a separate
function from `wakeup()` because sometimes the scheduler needs to send a wakeup
call while it's already holding the process table lock. So we're gonna assume
that every function that ever calls this is already holding it.

The implementation here is actually pretty simple now: we'll just iterate over
the process table and set every single process that's sleeping on channel
`chan` to `RUNNABLE`.
```c
static void wakeup1(void *chan)
{
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == SLEEPING && p->chan == chan) {
            p->state = RUNNABLE;
        }
    }
}
```

Now, there might be multiple processes sleeping on this channel, so this will
wake them all up. For some of those processes, this might be a spurious wakeup,
so again, we should always make sure to call `sleep()` in a loop that checks for
some condition to be satisfied. Even if multiple processes do have their
sleep conditions satisfied, they'll have to reacquire their original lock before
returning out of `sleep()`, so only one of them will do so and the others will
spin until the first one is done.

Why not just wake up the first process we find that's sleeping on `chan`? Then
we could avoid the extra overhead of a bunch of processes waking up, checking a
condition, and going back to sleep, or even spinning idly waiting to reacquire
the lock before returning. The issue is that the channels may not be unique, so
there's no way to know which of all the sleeping processes is the one whose
sleep condition has just been fulfilled. If we wake up the wrong process, it'll
just go back to sleep, but the right process didn't wake up, so that means we've
lost a wakeup call.

### exit

Okay, so we saw above that `kill()` doesn't really kill a process immediately;
it shirks that responsibility and lets `exit()` handle it instead... except
even `exit()` won't really fully kill a process. Whew, a process's death just
keeps getting dragged out forever, doesn't it? It's starting to feel like a
cheesy death scene in a tragedy; I bet the process is tired of suffering the
slings and arrows of outrageous fortune by now.

But it does make sense. Think about what we have to do in order to wrap up a
process and recycle its slot in the process table: we have to close out any open
files and reset its current working directory, free its kernel stack and its
entire page directory, then notify the parent that it's done running.

The trouble comes with freeing the kernel stack and process page directory. This
function runs in kernel mode, so while the user stack in the lower half of
memory will be unused now, the kernel stack is still needed in order to keep
executing the instructions for `exit()`. Also, with the exception of the times
when it's running the scheduling algorithm, the kernel uses the page directory
of the current process. The moment we free that page directory, the very next
memory access will be to an invalid page; the CPU would trigger an exception
then. That exception would eventually get routed to `exit()` again, except, oh
wait, we can't even run any instructions without generating another exception,
because the entire page directory and stack have been freed; that's a double
fault. So then the CPU would try to handle *that* exception, which would cause
the dreaded boogeyman of OS devs around the world: a triple fault. After a fault
triggers a second exception, which itself triggers a third exception, the CPU
just decides that the kernel in its current state doesn't have its shit together
enough to keep running, so it takes over and reboots the whole system. Oops.

Okay, so let's not do that. That means we can't free the kernel stack nor the
page directory until we're running on a different stack/page directory combo.
That could happen in `scheduler()` while we're using the page directory `kpgdir`,
or it could happen while we're running another process. xv6 does it while it's
running the parent process, in the `wait()` system call. If you haven't used
that in Linux before, `wait()` lets a parent process sleep until a child process
is done running. xv6 will use `wait()` to finish cleaning up after an exited
child process too.

Now, the very first process that starts running in xv6 (`initproc`, which loads
and runs the shell) obviously has no parent process, but that's okay because
that one should never exit as long as the system is up. So let's start this
function off by making sure that the process that's exiting isn't the initial
process.
```c
void exit(void)
{
    struct proc *curproc = myproc();
    if (curproc == initproc) {
        panic("init exiting");
    }
    // ...
}
```

Next we'll close all open files and clear the current working directory; again,
we haven't seen the file system functions used here, but we'll get to them soon.
```c
void exit(void)
{
    // ...

    // Close all open files
    for (int fd = 0; fd < NOFILE; fd++) {
        if (curproc->ofile[fd]) {
            fileclose(curproc->ofile[fd]);
            curproc->ofile[fd] = 0;
        }
    }

    // Clear the current working directory
    begin_op();
    iput(curproc->cwd);
    end_op();
    curproc->cwd = 0;

    // ...
}
```

Now we only have one thing left to do: notify the parent process that this
process has exited. If the parent process is currently sleeping in `wait()`,
then we'll need to wake it up. But maybe the parent process is currently in the
middle of executing other code before it gets to `wait()`; we don't want it to
miss the wakeup call... oh wait, but that's okay, remember? The implementations
of `sleep()` and `wakeup()`/`wakeup1()` guarantee that we can't miss a wakeup
call as long as we're holding the right lock; `wait()` will use the process
table lock for that. So let's acquire it now and send a wakeup call.
```c
void exit(void)
{
    // ...
    acquire(&ptable.lock);
    wakeup1(curproc->parent);
    // ...
}
```

Now, remember that a sleeping process needs to check some condition in a loop;
how can the parent process know that the child has exited? Hmm, okay, let's set
the child's state to `ZOMBIE`. That'll also prevent the scheduler from trying to
run it again.

Ah, but hang on a sec... what if the parent process has itself been killed, i.e.
the current process has been orphaned? (Again with the melodrama...) A process
can't run any more user code after `exit()`, so an undead parent process would
never get to call `wait()` to clean up after its children. In that case, we'd
have to find another process that could adopt a child.

So let's just solve that problem now: this process is about to shuffle off its
mortal coil, so let's figure out if it has any children and pass them off to
another process that can keep raising them as its own. But which process is
guaranteed to live long enough to clean up after those children once they die?
Ah, `initproc`, of course! That first process is immortal, so it should be able
to look after any children that this process might leave behind after it makes
its quietus with a bare bodkin.

So we'll iterate over the process table, looking for any processes with parent
process equal to `curproc`; if we find any, we'll have `initproc` adopt them.
If any of our now-abandoned children has already exited before we did, we'll
send a wakeup signal to `initproc` too in case it's sleeping in `wait()`.
```c
void exit(void)
{
    // ...
    for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->parent == curproc) {
            p->parent = initproc;
            if (p->state == ZOMBIE) {
                wakeup1(initproc);
            }
        }
    }
    // ...
}
```

Okay, now it's finally time for this process to find out what dreams may come in
that sleep of death. We'll set its state to `ZOMBIE` and context-switch into the
scheduler, never to return; if something goes wrong and the scheduler *does*
return, we'll panic in order to keep this function from returning into user code
again.
```c
void exit(void)
{
    // ...
    curproc->state = ZOMBIE;
    sched();

    panic("zombie exit");
}
```

### wait

Like we said above, this system call lets a parent process wait for a child
process to exit; it also cleans up after the child process has exited.

First, we don't even know if this process has any children, so we'll have to
check by iterating through the process table and checking each process's parent
to see if it matches the current process. If it does, then we'll check if it's a
zombie, in which case we can clean it up and return its process ID.

We should also deal with two edge cases: first, if the process has no children
at all, and second, if the process does have children but none of them are dead
yet. In the first case, we'll just return -1 to report failure; in the second
case we'll put the current process to sleep until one of its children exits. The
`sleep()` call means we'll have to do these checks inside an infinite loop.

Alright, let's get started by getting the current process and acquiring the
process table lock, then starting an infinite loop.
```c
int wait(void)
{
    struct proc *curproc = myproc();

    acquire(&ptable.lock);

    for (;;) {
        // ...
    }
}
```

Inside the loop, we'll use a variable `havekids` as a boolean to track whether
we've found any child processes. Then we can iterate over the process table,
skipping any processes for which the current process is not the parent. If we
find any children, we'll set `havekids` to 1.
```c
int wait(void)
{
    // ...
    for (;;) {
        int havekids = 0;

        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            if (p->parent != curproc) {
                continue;
            }
            havekids = 1;
            // ...
        }
        // ...
    }
}
```

If we did find a child process, we should check if it's a zombie, in which case
it's time to finish its clean-up. That means freeing its kernel stack and its
page directory and recycling its `struct proc` so that it can be reallocated to
another process later on.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        for (struct proc *p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
            // ...
            if (p->state == ZOMBIE) {
                int pid = p->pid;

                // Free child's kernel stack
                kfree(p->kstack);
                p->kstack = 0;

                // Free child's page directory
                freevm(p->pgdir);

                // Recycle child's struct proc
                p->pid = 0;
                p->parent = 0;
                p->name[0] = 0;
                p->killed = 0;
                p->state = UNUSED;

                release(&ptable.lock);
                return pid;
            }
        }
        // ...
    }
}
```

Now, if `havekids` is still zero by the time we finish the for loop, that means
the process doesn't have any children, so we should report failure. We'll also
check if the process has been marked as killed in the meantime.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        if (!havekids || curproc->killed) {
            release(&ptable.lock);
            return -1;
        }
        // ...
    }
}
```

Finally, if it *does* have children, but none of them have exited yet, we'll put
the process to sleep. It'll get woken up when a child exits, at which point
it'll restart the outer for loop at the top and start looking through the
process table again.
```c
int wait(void)
{
    // ...
    for (;;) {
        // ...
        sleep(curproc, &ptable.lock);
    }
}
```

## Summary

By now, we've looked at a good chunk of the system calls available in xv6. These
system calls wrap up the mechanisms that xv6 uses to create and exit processes
with `fork()`, `kill()`, `exit()`, and `wait()`, and introduced `sleep()` and
`wakeup()` as a means for (limited) inter-process communication.

So what's left now? The rest of the kernel code we're gonna look at will just
focus on communicating with various hardware devices like the serial port,
console, and keyboard. Those drivers are relatively short, but there's one
device that will require a lot more work: the disk. Storing files on disk and
making sure they persist across reboots require careful planning, and making
files conveniently accessible to users requires an entire system of abstractions
layered on top of each other, along with a whole host of file-related system
calls.
