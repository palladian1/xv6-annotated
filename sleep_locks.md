# Sleep Locks

We've used plenty of spin-locks, and a previous post looked at their
implementation in xv6. Spin-locks have pretty harsh performance costs: a process
that's waiting to acquire a lock will just spin idly in a while loop, wasting
valuable CPU time that could be used to run other processes. So far, we've only
seen locks for kernel resources like the process table, page allocator, and
console, for which all operations should be relatively fast, on the order of a
few dozen CPU cycles at most.

Now it's time to look at the disk driver and file system implementation, and
we'll need some locks there too. But disk operations are *slow* -- reading from
and writing to disk might take milliseconds, which is a literal eternity for a
CPU. Imagine a process hogging a spin-lock for the disk while other processes
spin around and around waiting *forever* for the disk to finish writing. It
would be an enormous waste!

Spin-locks were the best we could do at the time, since we didn't have any
infrastructure to support more complex locks, but now we really do need a better
alternative. We also have some more kernel building blocks in place relating to
processes, including a bunch of system calls.

For example, we've seen the `sleep()` and `wakeup()` system calls, which let a
process give up the CPU until some condition is met. Well, hang on a second --
what if that condition is that a lock is free to acquire? Then a process could
sleep while another process holds the lock, and wake up when it's ready to be
acquired; that would let other processes run instead of forcing a process to
spin and spin. xv6 calls these *sleep-locks*, and it's time to find out how they
work.

## sleeplock.h

If we want a process holding a sleep-lock to give up the processor in the middle
of a critical section, then sleep-locks have to work well when held across
context switches. They also have to leave interrupts enabled. This couldn't
happen with spin-locks: it was important that they disable interrupts to prevent
deadlocks and ensure a kernel thread can't get rescheduled in the middle of
updating some important data structure.

Leaving interrupts on adds some extra challenges. First, we have to make sure
the lock can still be acquired atomically; second, we have to make sure that any
operations in the critical section can safely resume after being interrupted.

Let's solve the first problem: how can we make sure a sleep-lock will always be
acquired atomically? Well, if we want to do something atomically, we already
have a solution: spin-locks! So rather than reinventing the wheel, we'll just
make each sleep-lock a two-tiered deal with a spin-lock to protect its
acquisition.

We'll use a `locked` field just like the one all spin-locks have, but then we'll
add a spin-lock to protect it. We'll also make debugging a little easier by
adding a name for the lock and a field for a PID to identify which process is
holding it.
```c
struct sleeplock {
    uint locked;
    struct spinlock lk;
    char *name;
    int pid;
};
```

## sleeplock.c

### initsleeplock

We can initialize a sleep-lock by initializing its guard spin-lock, then adding
a name for it, setting `locked` to false, and the `pid` field to zero.
```c
void initsleeplock(struct sleeplock *lk, char *name)
{
    initlock(&lk->lk, "sleep lock");
    lk->name = name;
    lk->locked = 0;
    lk->pid = 0;
}
```

### acquiresleep

In order to make sure sleep-lock acquisition is atomic, we'll bookend this
function by acquiring and releasing a spin-lock. This will also make sure that
interrupts are disabled during this function but re-enabled when it's done. It
does add some overheard in the form of spinning until this lock is free, but the
code here should be relatively short and fast to execute. What we really want is
to avoid spinning once the sleep-lock is acquired, i.e. spinning *after* this
function is done. So we'll tolerate a little waste here.
```c
void acquiresleep(struct sleeplock *lk)
{
    acquire(&lk->lk);
    // ...
    release(&lk->lk);
}
```

Okay, now we have to do the actual acquisition. We said above that we'd use the
`sleep()` function to avoid wasting processor time. Hopefully you remember one
important detail about `sleep()`: it must always be called inside a while loop
in order to make sure that we don't miss any wakeup calls. So let's check if the
sleep-lock is already being held and go to sleep if it is. We'll need a channel
and a lock for `sleep()` to release, so let's use the pointer to this lock `lk`
as the channel, and the outer spin-lock `lk->lk` as the lock to be released.
```c
void acquiresleep(struct sleeplock *lk)
{
    // ...
    while (lk->locked) {
        sleep(lk, &lk->lk);
    }
    // ...
}
```
It's important to keep the two locks separate in your head right now: `lk` is
the sleep-lock, and `lk->lk` is the spin-lock it uses to protect the sleep-lock's
acquisition. Note that we're checking `lk->locked` here, *not* the spin-lock
`lk->lk` -- this process is already holding `lk->lk`, but we need to acquire
`lk` itself by updating `lk->locked`. Phew, try saying that ten times fast.

Now the process will go to sleep and yield the CPU until the sleep-lock is free.
If multiple processes are sleeping waiting on the same sleep-lock, they will all
wake up at the same time, but all of them have to reacquire `lk->lk` before
returning from sleep, so only one will get to return here and complete the
sleep-lock acquisition. The others will spin a bit longer, then return here only
to find that `lk->locked` is already being held by another process, so the while
loop will put them to sleep again.

Once the sleep-lock is free, the process can exit the while loop and claim the
sleep-lock for itself.
```c
void acquiresleep(struct sleeplock *lk)
{
    // ...
    lk->locked = 1;
    lk->pid = myproc()->pid;
    // ...
}
```
We don't need fancy atomic operations like `xchg` anymore, since the guarding
spin-lock has already made sure that interrupts are disabled and all operations
are effectively atomic. So that's all we need! Now we just release the spin-lock
and return.

### releasesleep

Now that we've seen how a process acquires a sleep-lock, releasing it is easy,
we just do the opposite. We'll set `lk->locked` to zero and clear the `lk->pid`
field. And what's the opposite of `sleep()`? Well, `wakeup()`, of course! That
will check whether there are any processes sleeping on this channel and let them
know they can attempt to acquire the sleep-lock now.
```c
void releasesleep(struct sleeplock *lk)
{
    acquire(&lk->lk);

    lk->locked = 0;
    lk->pid = 0;

    wakeup(lk);

    release(&lk->lk);
}
```

### holdingsleep

This function is even more simple: it just checks whether a sleep-lock is being
held, and if so, whether it's being held by the current process. The first is
done by just checking `lk->locked`; the second is done by checking that `lk->pid`
matches the current process's PID. The result is a boolean stored in a temporary
variable so we can release the guarding spin-lock before returning the result.
```c
int holdingsleep(struct sleeplock *lk)
{
    acquire(&lk->lk);

    int r = lk->locked && (lk->pid == myproc()->pid);

    release(&lk->lk);
    return r;
}
```

## Summary

Okay, that wasn't too bad! It makes sense why we couldn't use sleep-locks in a
kernel without system calls like `sleep()` and `wakeup()`. But xv6 already has
those, so why not use them everywhere? If sleep-locks really do cut down on the
wasted CPU time, can we just go back and replace all the spin-locks with
sleep-locks? Then the only use for spin-locks would be as a guard for the more-
sophisticated sleep-locks.

Hold your horses! It's not that easy. Sleep-locks leave interrupts enabled, so
they can't be used in interrupt handler functions, or inside a critical section
where a spin-lock is being used, since interrupts will be disabled (though spin-
locks can be used inside sleep-lock critical sections). They also can't be used
by kernel threads like the scheduler, since those aren't processes and thus
can't be put to sleep.

Finally, there are some situations in which a sleep-lock might actually add
*more* overhead than a spin-lock: it takes some time to put a process to sleep,
schedule another process, send a wakeup call, schedule the first process again,
and so on, and the process will hold the sleep-lock the entire time. If another
process is waiting on the sleep-lock, it might actually end up waiting longer
than with a spin-lock, although it'll wait in a sleeping state instead of a
running state where it just spins in a loop.

Additionally, sleep-locks can only be used when it's safe to interrupt a process
in the middle of a critical section and wake it up later. Sure, no other process
can acquire the sleep-lock in the meantime, but it's still not great for time-
sensitive operations like getting the current number of `ticks`.

So sleep-locks are great, but their applications are more limited than spin-
locks. The perfect use for them is when a process needs to complete an operation
atomically, but that operation itself might take a very long time. A great
example of that is disk I/O, and we'll see next how xv6 puts them to use in its
file system implementation.
