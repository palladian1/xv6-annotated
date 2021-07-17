# Detour: Spin-Locks

So I know I said I wasn't expecting you to have finished the OSTEP section on
concurrency, but xv6 uses locks all over the place, so we're gonna have to get
comfortable with them right away. Luckily, xv6 primarily uses spin-locks, which
are super simple and work on bare metal; a lot of the more complex/more awesome
locks that OSTEP talks about require an OS beneath them.

I'll give a brief intro to concurrency first in case you haven't made it to that
part in OSTEP; then we'll turn to the spin-lock implementation in xv6.

## A Very Brief, Poor-Man's Intro to Concurrency

TL;DR: Concurrency is your worst nightmare. It'll cause bugs in the places where
you least expect it, and they won't even be consistent: your code might work 95%
of the time, but every once in a while it'll randomly fail and you'll have no
idea why. The good news: xv6 handles it in a super-simple way, so we'll get to
appreciate it as we go along. If you're like me, you might also see the code use
locks when you wouldn't have thought they were needed, and then you'll come to
appreciate just how clever the xv6 authors are.

First off, stop reading this and go watch the discussion of data races and locks
in [the last few minutes of the CS 50 2021 lecture on SQL](https://www.youtube.com/watch?v=LzElj46saa8&t=8762s).
I'm serious, go watch it right now; this post will still be here.

Okay, I'm gonna assume you've seen it now; you should have a decent sense of the
main issues with data races and how locks solve them. But the CS 50 lecture
skipped some details about locks: (1) what Brian (the TA) does when he finds a
locked fridge, (2) how locks are implemented in code, and (3) deadlocks.

### What Does Brian Do?

Let's say process `david` is running on one thread, and it needs to use some
resource (a global variable maybe, or an I/O device like the disk or console)
that other threads might want to use too, so `david` acquires the lock for that
resource. Then process `brian` comes along and wants to use the same resource at
the same time. This could cause a data race, but luckily we've thought ahead and
used a lock, so `brian` can't access it until `david` is done with it and
releases the lock.

First of all, we better hope `david` remembers to release the lock; otherwise
`brian` (and all other processes, even the kernel) will *never* be able to use
that resource. But assuming we're smart and remembered to release it, what does
the `brian` process do in the meantime?

Well, maybe `brian` has some other work to do that he can get started on in the
meantime. But what would that mean for an OS? How would we know, in general,
whether the lines of code that follow the use of a shared resource can be safely
executed if we haven't used that resource yet? That sounds impossible to figure
out without knowing ahead of time what the resource is and how it's used, so
let's just go ahead and skip that idea.

Another option that's actually used often in the real world is for `brian` to
stop trying and go to sleep. Maybe he can put a note on himself asking `david`
to wake him up when he gets back with the milk. So in code, that might look like
`brian` signaling the OS and letting it run a different process until the lock
is released. That sounds nice and all, but at this early stage in our kernel, we
don't even have processes or a scheduler yet, let alone a notion of sleeping.

Okay, another option: what if `brian` just spins around in circles, or twiddles
his thumbs, or does jumping jacks or whatever until `david` releases the lock?
In code, that means looping over and over forever until the lock is released.
That would be horribly inefficient; think of all the CPU time wasted when one
process just loops over and over again while another process does something slow
while holding a lock! But it's also the approach that xv6 is gonna take, because
at the end of the day, our kernel is still in baby stages and beggars can't be
choosers. So xv6 uses *spin-locks* with loops that only stop when we acquire a
lock.

This means we should be careful when using locks to acquire them only at the
last possible moment when they're absolutely needed, and release them as soon as
they're no longer required, in order to limit the amount of wasted CPU cycles.

### Implementing Locks

We can implement locks as a simple boolean variable: if it's true, then someone
else is using the resource behind the lock. If it's false, then it's unused and
you can go ahead and take it. So an `acquire()` function sets the lock to `true`
and a `release()` function sets it back to `false`. Done!

But it's not so simple: there's actually a race condition hidden in the very
idea of a lock. Think about it for a second: a lock protects some shared
resource, right? And a shared resource is something that more than one process
wants to use? But a lock is itself a thing that more than one process wants to
use... so we haven't actually gotten rid of the race condition. (FLIPS TABLE.)

We have another Catch-22 on our hands, but this time we can't get rid of it with
a clever software trick like we did with the `entrypgdir`. The issue is that no
matter how well we write our code, it will always require more than one step:
first we have to check whether the lock is `true`, then we have to set it to
`true`. But if someone else is doing the same thing at the same time, our
instructions might get executed in parallel and then we'd both acquire the lock
at the same time -> RACE CONDITION.

The solution will require hardware support, using *atomic* instructions -- these
are hardware instructions that are indivisible; no other code can execute in
between ours. One example is the x86 instruction `xchg`, which atomically reads
a value from memory, updates it to a new value, and returns the old value.

Now we're good! A lock can still be a boolean variable but now `acquire` has to
use `xchg`: it should get the old value while simultaneously updating it to
`true`.

Atomic instructions have more overhead than regular ones, so we should only use
them when they're required, like in locks, but otherwise we can stick to the
regular instructions we've always used.

There's one other detail we should be careful about: a lot of the locks in xv6
protect resources that are needed by both interrupt handlers and kernel or user
code. For example, we might use a process table lock to protect the list of all
currently running processes; suppose some kernel code has acquired the lock in
order to run a new process. What happens if a timer interrupt goes off at that
moment? The timer interrupt handler function might need to acquire the lock in
order to switch processes, but it's already being held by the kernel thread. But
the timer interrupt might take priority over the kernel thread and refuse to
return to the kernel until it finishes executing. The result: that CPU comes to
a total halt as the timer interrupt handler function spins forever, never to get
the lock it so desperately needs to move on. So sad. :(

xv6 avoids this issue in a really simple way: every time we acquire a lock,
we'll just disable interrupts altogether. Problem solved: now a thread can't get
interrupted until it's done using the lock and releases it. This does mean that
a process which grabs locks often might stick around longer than it should,
since we won't have timer interrupts to tell the scheduler to swap it out with
another process, but we're just gonna cross our fingers and hope that doesn't
happen too often.

### Deadlocks

The last concurrency issue we need to be aware of is the problem of deadlocks.
Suppose two threads each need locks A and B; this happens often, e.g. when
loading a user program the kernel will need to hold a lock for the disk and
another for the process table, or a process might be reading from disk and
printing to the console at the same time.

Suppose they're running at the same time, and one process acquires lock A while
the other one acquires lock B. If they each need the other lock to keep going,
they'd spin forever waiting for it. This is a deadlock.

The way to avoid these is to make sure that, if we use more than one lock, we
*always* acquire them in the same order. That way, one process would acquire
lock A, the second one would be unable to acquire it and would spin, then the
first process acquires lock B with no issues. When it's done, it releases both
locks and the second process can continue.

This can get complicated though: if we ever acquire a lock in a function, we'd
have to check any functions that that function calls to see whether they use any
locks, and so on. If they do, and if the order conflicts with another chain of
function calls, we'd have to refactor the code until the orders match. xv6 has
been carefully written so that the lock acquisition order is always consistent.

## spinlock.c

xv6's spin-locks are set up as a `struct spinlock`, defined in
[spinlock.h](https://github.com/mit-pdos/xv6-public/blob/master/spinlock.h). The
`locked` field acts as the boolean variable to determine whether the lock is
held; the other fields are for debugging, since we can expect concurrency issues
to be the one of the most common causes of bugs in the kernel code because,
again, concurrency is your worst nightmare.

Note that `locked` is an `unsigned int` instead of a `bool`; C requires the
standard library header *stdbool.h* in order to use the `bool` type, but on
bare metal we can't assume we have a standard library to use.

### initlock

```c
void initlock(struct spinlock *lk, char *name)
{
    lk->name = name;
    lk->locked = 0;
    lk->cpu = 0;
}
```

This function is pretty straightforward; it just stores the string `name` in
the lock and starts it off as unlocked; the `cpu` field is 0 because no CPU is
holding it yet. Next.

### pushcli and popcli

For reasons mentioned above, we need to disable interrupts whenever we're using
a lock and re-enable them when we release a lock. But if we're not careful, we
could end up enabling interrupts too early when we release one lock while still
holding another; or if interrupts were already disabled when we acquired a lock,
we could unintentionally re-enable them upon releasing it.

xv6 uses paired functions `pushcli()` and `popcli()`.
```c
void pushcli(void)
{
    int eflags = readeflags();
    cli();
    if (mycpu()->ncli == 0) {
        mycpu()->intena = eflags & FL_IF;
    }
    mycpu()->ncli += 1;
}
```
`readeflags()` is a C wrapper for some x86 assembly code that reads from the
`eflags` register; the 9th bit is the interrupt flag, which is set whenever
interrupts are enabled. `cli` is another x86 instruction that clears that flag,
thus disabling interrupts.

`mycpu()` returns a pointer to a `struct cpu` with information about the CPU
running this code; we'll go over these when we talk about processes; here we
increment the `ncli` field in every call to `pushcli()`. If this is the first
call, we save the value of the interrupt flag in the `intena` field.

```c
void popcli(void)
{
    if (readeflags() & FL_IF) {
        panic("popcli - interruptible");
    }
    if (--mycpu()->ncli < 0) {
        panic("popcli");
    }
    if (mycpu()->ncli == 0 && mycpu()->intena) {
        sti();
    }
}
```
`popcli()` first checks to make sure interrupts aren't already enabled and we're
not popping without having pushed. Then it decrements the `ncli` field of the
`struct cpu` for this CPU. If this is the last call to `popcli()`, it checks the
`intena` field; if it was set (i.e., interrupts were enabled before the first
`popcli()`), then it enables interrupts again.

Check out how these two functions are carefully written so that they're matched:
it takes two calls to `popcli()` to undo two calls to `pushcli()`. Also, if
interrupts were already off before the first call to `pushcli()`, they'll stay
off after the last `popcli()`. Pretty neat, right?

### holding

This function checks whether this CPU is holding the lock.
```c
int holding(struct spinlock *lock)
{
    pushcli();
    int r = lock->locked && lock->cpu == mycpu();
    popcli();
    return r;
}
```
Not much to talk about here; it just checks (inside calls to `pushcli()` and
`popcli()`) whether the lock is being held and this is the CPU holding it. If
both conditions are true it'll return 1; otherwise 0.

### acquire

The first step in this function is to disable interrupts to avoid deadlocks. We
also make sure we're not already holding the lock; otherwise we'd deadlock
ourselves.
```c
void acquire(struct spinlock *lk)
{
    pushcli();

    if (holding(lk)) {
        panic("acquire");
    }

    // ....
}
```

Next up, we've gotta acquire the lock using the atomic `xchg` instruction,
defined in [x86.h](https://github.com/mit-pdos/xv6-public/blob/master/x86.h).
Like we said before, the trick is to atomically set `locked`
to 1 while returning the old value. If the returned old value is 1, that
means it was already 1 before we got to it, so it's currently being held and we
can't acquire it yet -- gotta spin. But if the returned old value is 0, that
means the lock was free before we got to it, and our `xchg` just updated it to
1, so we've successfully acquired it. No other instruction can occur between
checking the old value and updating it to the new one, so we can be confident
that no one else will be holding the lock at the same time.
```c
void acquire(struct spinlock *lk)
{
    // ...
    while (xchg(&lk->locked, 1) != 0)
        ;
    // ...
}
```

We do have to be careful about one other thing: compiler optimizations can get
pretty wild nowadays, so the order of code on the page isn't necessarily the
order it'll get compiled to or executed in. This is a critical section of code,
so we need to make sure acquiring the lock forms a barrier between the code that
comes before it and the code after it so any reordering doesn't cross the lock
acquisition point. We can do that with a special compiler instruction:
```c
void acquire(struct spinlock *lk)
{
    // ...
    __sync_synchronize();
    // ...
}
```

Finally, we'll record some info about the CPU and process holding the lock for
debugging purposes. Don't worry about `mycpu()` for now, but we'll talk about
`getcallerpcs()` below.
```c
void acquire(struct spinlock *lk)
{
    // ...
    lk->cpu = mycpu();
    getcallerpcs(&lk, lk->pcs);
}
```

### release

Releasing a lock is a little easier than acquiring it: to acquire it, we need to
check whether it's already held and update its value, with both steps together
as an atomic instruction. To release it, we only have to set the value to false.
That's only one instruction, so it's automatically atomic!

Well, almost, but not quite. The compiler works some serious magic behind the
scenes, so there's no guarantee that a single C operation like `lk->locked = 0`
will actually get compiled down to a single assembly instruction. So we're gonna
have to make sure it does by writing it directly in assembly.

We start off by making sure we are already holding the lock before releasing a
lock held by someone else. Then we clear the debug info stored in the lock, and
tell the compiler and processor not to reorder code past the lock release.
```c
void release(struct spinlock *lk)
{
    if (!holding(lk)) {
        panic("release");
    }

    lk->pcs[0] = 0;
    lk->cpu = 0;

    __sync_synchronize();

    // ...
}
```

Next we need to release the lock, i.e. an assembly instruction equivalent to
`lk->locked = 0` in C. C allows in-line assembly code using the `asm` keyword.
We mark it as `volatile`, which prevents the compiler from optimizing the write
away and ensures it'll get written to memory. Finally, we call `popcli()` to
enable interrupts again.
```c
void release(struct spinlock *lk)
{
    // ...
    asm volatile("movl $0, %0" : "+m" (lk->locked) : );

    popcli();
}
```

### getcallerpcs

This function exists to store information about the current process in the lock
for use in debugging. In particular, we want to record the program counters of
the last 10 functions on the call stack so we can try to figure out which
functions were called in which order when concurrency issues inevitably bring
our world crashing down with data races, or to a grinding halt with deadlocks.

In order to get the program counters, we're gonna have to know a bit about how
x86 handles function calls. The `%eip` register (or instruction pointer) holds
the program counter, which tracks the next instruction to be executed. The
`%ebp` register (or base pointer) holds the address of the base of the stack
(i.e., its highest address, since it grows down).

When a function gets called all its arguments are pushed on the stack in reverse
order, so that the first argument is at the top (lowest address) of the stack.
Then the previous function's `%eip` is pushed on the stack, followed by its
`%ebp`:
```
<- low addresses                                               high addresses ->
...  [new function's data]  [old %ebp]  [old %eip]  [new arg1]  [new arg2]  ...
<- top of stack                                               bottom of stack ->
```

Anyway, the point is that if we have the address of the first argument to the
current function, then we can recover the contents of the previous function's
`%ebp` and `%eip` registers: `%eip` is one spot below it on the stack and `%ebp`
is two spots below it.
```c
void getcallerpcs(void *v, uint pcs[])
{
    uint *ebp = (uint *) v - 2;
    // ...
}
```
Note the type casts here -- `v` is a pointer to the first argument, which can be
of any type and size, so we use a `void *`. But both of the `%eip` and `%ebp`
registers hold 32-bit pointers, so `ebp` is declared as a pointer to a `uint`
(a type alias for `unsigned int`, remember?), which makes the pointer arithmetic
work out nicely so that subtracting 2 returns a pointer to the right spot on the
stack.

Now, what we really want is the program counter `%eip`, not the pointer to the
stack base `%ebp`. But we can use the address of `%ebp` to make sure we haven't
gone too far back in the function call history. Remember, we wanna get the
program counters for the last 10 functions in the call stack, then save them in
the `pcs` array.
```c
void getcallerpcs(void *v, uint pcs[])
{
    // ...
    int i;
    for (i = 0; i < 10; i++) {
        // Stop if the %ebp pointer is null or out of range
        if (ebp == 0 || ebp < (uint *) KERNBASE || ebp == (uint *) 0xffffffff) {
            break;
        }
        pcs[i] = ebp[1];
        ebp = (uint *) ebp[0];
    }
    // ...
}
```
Let's talk about those last two lines: the `ebp` pointer in the code holds the
location of the saved `%ebp` register, so `ebp[0]` is the value at that address
(i.e., the actual value of the saved `%ebp` register) and `ebp[1]` is the value
stored one spot above that, i.e. the value of the saved `%eip` register. So
each iteration of the loop will get one `%eip` and store it in a `pcs` entry.

Then we update `ebp` to the actual value at the address it points to, which
means `ebp` will now point to the address of the saved `%ebp` register for the
function one step further back in the call chain. Okay sorry, I know that's
confusing, but basically each iteration of the for loop moves us back to the
function that called this function, then the function that called that one, and
so on.

Okay, whew. So what happens if we break out of the for loop early because we
went all the way back in the call stack? The other entries of `pcs` might hold
some garbage values, so let's just make them null pointers so we know to ignore
them when debugging.
```c
void getcallerpcs(void *v, uint pcs[])
{
    // ...
    for (; i < 10; i++) {
        pcs[i] = 0;
    }
}
```
One last little trick: the previous for loop declared the loop variable `i`
before the loop -- this means `i` will be in scope for the rest of the function
body. If it had been declare inside the for loop like `for (int i = 0; ...)`, it
would fall out of scope at the end of the loop. So we can keep using the same
`i` in this second for loop (without an initialization statement) and know it'll
hold the value it had after finishing the first for loop. If we finished all the
iterations, that value will be 10; otherwise it'll be less. So we use that to
clear any remaining entries of `pcs`.

## Summary

You'll learn to hate concurrency issues in C; newer languages like Rust make
data races a thing of the past, though deadlocks can still rear their ugly
heads. But for now, the xv6 authors have done all the dirty work for us, so we
can just sit back and watch. Note, though, that even the xv6 authors say it's
totally possible that something has slipped past them and the thousands of other
students and instructors that have looked at xv6, so it's probable that xv6
still has some lingering race conditions. See, even the masters struggle with
it. -_-

Anyway, we saw that locks have to be implemented with hardware support using
atomic instructions. C and most languages provide high-level atomics that real-
world operating systems use, but the point of xv6 is elegance in simplicity, not
being a total show-off, so the xv6 spin-locks just use the basic `xchg`.

We took this detour into spin-locks to make sure we all understand some basic
details because we're gonna be seeing a lot of them in the rest of the kernel
code. They're inefficient (because the processor just spins around waiting for
the lock to be released, WHEEEEE), but we gotta make do with the machinery we've
built up so far. xv6 will also use some fancier locks called sleep-locks, but
we'll cross that bridge when we get to it.
