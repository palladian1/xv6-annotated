# System Calls: Routing

We said in the last post that system calls are the primary means for user
processes to request some action by the kernel; system calls mediate processes'
access to hardware resources.

If a user process wants to generate a system call, it starts a trap with the
trap number for system calls. Then it identifies which of the various xv6 system
calls it wants to do and passes any required arguments. The processor will then
handle the trap instruction using the code we saw in the last post. Eventually,
it'll get to the `trap()` function, which will recognize the trap number as a
system call and pass it on to the `syscall()` function.

`syscall()` is itself a routing function like `trap()`; it'll figure out which
system call the process created and redirect it again to the appropriate kernel
code.

## syscall.c

All system calls use the same trap number: 64, or `T_SYSCALL`, but xv6 has
multiple system calls, so we need another number for a process to identify which
system call it wants to run. The convention on x86 is to use a system call
number which the calling process should put in the `%eax` register, which
usually holds return values. Then the kernel's handler function (here,
`syscall()`) can just check `%eax` to figure out which system call to run. The
system call numbers are defined in
[syscall.h](https://github.com/mit-pdos/xv6-public/blob/master/syscall.h). There
you can see that, e.g. `SYS_fork` is defined as 1, `SYS_exit` is 2, and so on.

All the system call functions are defined in other files, so we'll have to
import their declarations with the `extern` keyword:
```c
// ...
extern int sys_chdir(void);
extern int sys_close(void);
extern int sys_dup(void);
extern int sys_exec(void);
extern int sys_exit(void);
extern int sys_fork(void);
extern int sys_fstat(void);
extern int sys_getpid(void);
extern int sys_kill(void);
extern int sys_link(void);
extern int sys_mkdir(void);
extern int sys_mknod(void);
extern int sys_open(void);
extern int sys_pipe(void);
extern int sys_read(void);
extern int sys_sbrk(void);
extern int sys_sleep(void);
extern int sys_unlink(void);
extern int sys_wait(void);
extern int sys_write(void);
extern int sys_uptime(void);
// ...
```

Now we've got the numbers and the functions. Note that the numbers start with
uppercase `SYS_` and the functions start with lowercase `sys_`, so make sure
your kernel hacking adventures don't do anything like `SYS_fork()`; use
`sys_fork()` instead.

We'll also need a way to map the numbers to those system call functions so that
`syscall()` can call the right one depending on the number. We could use another
`switch` statement like we did in `trap()`, but there are 21 system calls here,
so that would get pretty long; also, each number will just call the specific
function, unlike the different trap numbers which required different responses
(e.g., the timer interrupt trap number didn't call any function at all). xv6
does something else this time that's much simpler and more elegant, but it uses
some slightly-obscure C features, so we'll go over it carefully.

Remember function pointers from way back in the boot loader? Functions are just
a set of instructions in order, loaded somewhere in the kernel's code segment,
so C lets us use the function's name as a pointer to the beginning of its code
in memory. So if we have a C function like `int func(char c)`, then `func` is
its function pointer. We could even assign it to a variable; that variable's
type would be a pointer to a function of argument type `char` and return type
`int`; then we could call the function using the new pointer too. Here's an
example that would print "Match!" to the screen:
```c
int m = func('a');

int (*func_ptr)(char) = &func;
int n = (*func_ptr)('a');

if (m == n) {
    printf("Match!\n");
}
```

So instead of a big old `switch` statement, the `syscall()` function will use a
static, global array of pointers to all the system call functions we just
imported above. (Remember that the `static` keyword in front of a variable means
it always occupies the same fixed place in memory.) It'll work because all the
functions have the same argument type (`void`) and return type (`int`), so their
pointers all have the same type and can fit inside a single array. Then we can
get the right function by just using the system call number to index into the
array of function pointers.

Now, we'd have to be super careful to add the function pointers into the array
in the right order so that the indices match up. Even worse, there is no system
call with number zero, so we'd have to skip that entry of the array. This could
get complicated. Luckily, even though humans are bad at this kind of thing,
computers are *really* good at it. So instead of trying to line them up by hand,
we can use the array notation from `procdump()` in the post on processes where
we specified the value of each entry of an array like this with the index in
square brackets, like this:
```c
int arr[] = { [2] 5, [0] 1, [4] -2 };
```
The C compiler will use the indices we wrote there to figure out that the array
needs 5 entries (indices 0 to 4), and entry 0 is 1, entry 2 is 5, and entry 4 is
-2. Entries 1 and 3 will just be initialized to zero.

So at the end of the day, our array of pointers to system call functions looks
like this:
```c
// ...
static int (*syscalls[])(void) = {
    [SYS_fork]      sys_fork,
    [SYS_exit]      sys_exit,
    [SYS_wait]      sys_wait,
    [SYS_pipe]      sys_pipe,
    [SYS_read]      sys_read,
    [SYS_kill]      sys_kill,
    [SYS_exec]      sys_exec,
    [SYS_fstat]     sys_fstat,
    [SYS_chdir]     sys_chdir,
    [SYS_dup]       sys_dup,
    [SYS_getpid]    sys_getpid,
    [SYS_sbrk]      sys_sbrk,
    [SYS_sleep]     sys_sleep,
    [SYS_uptime]    sys_uptime,
    [SYS_open]      sys_open,
    [SYS_write]     sys_write,
    [SYS_mknod]     sys_mknod,
    [SYS_unlink]    sys_unlink,
    [SYS_link]      sys_link,
    [SYS_mkdir]     sys_mkdir,
    [SYS_close]     sys_close,
};
// ...
```

Okay great, now we're ready to route system calls to the right function.

### syscall

The first thing we need to do is get the system call number so we can figure out
which function to call. We said above that the x86 convention is to store it in
the `%eax` register, but we might have a problem: by the time we get to
`syscall()`, the processor has already executed the code in the trap handler
function for trap number `T_SYSCALL`, which sent it to `alltraps()`, which
replaced all the register contents with those of `trap()`, so the system call
number is probably long gone from `%eax`.

But wait, all is not lost! `alltraps()` saved all the registers in a
`struct trapframe` for the current process. So we can just read the value of
`%eax` from there. Whew, that was some good forward-thinking.
```c
void syscall(void)
{
    struct proc *curproc = myproc();
    int num = curproc->tf->eax;
    // ...
}
```

Now we just need to do one more thing: call the function that corresponds to
that number. We're gonna use the array of function pointers above, but we have
to be careful: this number was given to us by a user process. A malicious user
process might pass in an invalid number in the hopes of getting the kernel to
carry out some undefined behavior which might lead to an easy exploit. So in
order to keep up good security practices, the kernel should *always* distrust
anything originating from user code and handle it carefully, preferably with
three-inch-thick lead-lined gloves. So let's think about it: what might go
wrong?

First of all, any entries that weren't explicitly initialized above (including
the 0 entry) will have been automatically initialized to zero, i.e. a null
pointer. Also, a number that's bigger than the highest system call number will
make us do an out-of-bounds read from the array, thus possibly executing some
arbitrary kernel code that's stored after the array in memory. So we should
check that (1) the number is greater than 0, (2) it's smaller than the number of
elements in the array, and (3) the entry it points to is not a null pointer.

Finally, the `%eax` register is usually used in x86 to store return values, so
we'll put the return value of the system call function there. If any of the
above checks failed, we'll just print a message to the console and return -1 to
indicate failure.
```c
void syscall(void)
{
    // ...
    if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        curproc->tf->eax = syscalls[num]();
    } else {
        // Invalid system call number
        cprintf("%d %s: unknown sys call %d\n", curproc->pid, curproc->name, num);
        curproc->tf->eax = -1;
    }
}
```

The system call handler function will store its return value in `%eax`; after
that, `syscall()` will return to the line below where it was called in `trap()`.
After executing the rest of the code there, `trap()` will return into `trapret()`,
which ends with an `iret` (interrupt return) instruction to tell the processor
to switch to user mode and resume executing the process's code.

### fetchint

Take a look at the `sys_` functions we imported above: they all have argument
type `void`. But if you think about it, many system calls need an argument: for
example, `open()` needs to know which file to open, `chdir()` needs to know
which directory to open, `kill()` needs a PID to know which process to kill,
etc. So why did we make them all have argument type `void`?

The trouble is that until we get the system call number in `syscall()` above, we
have no way of knowing which function we'll need. And each function takes
arguments with different types, e.g. `open()` might need a string for the file
to open but `kill()` might need an integer for the PID. So there's no way for
the kernel to know which arguments to expect in `syscall()`, even though the
arguments were already pushed on the stack. The task of recovering the arguments
from the stack will have to fall to each of the `sys_` functions. But let's go
ahead and make their lives a little easier by setting up some nice helper
functions now.

The system call functions might take integers, strings, or pointers, so we'll
need functions to fetch each of those types. `fetchint()` is one example; it
takes a user virtual address (an integer argument's location in memory) and a
pointer to an integer where we can store the integer we find. Then it returns 0
if it was able to find it, or -1 if it failed.

Just like `syscall()` above, we need to treat anything passed from user space
with extreme caution. A user process that tries to read or write memory outside
its address space will cause a segmentation fault or page fault and be killed,
but the kernel has free reign over memory, so a malicious process might try to
trick the kernel into doing that *for* it by putting its "argument" outside of
the user's address space. So we have to start by checking that the entire 4
bytes of the integer is inside the process's address space.
```c
int fetchint(uint addr, int *ip)
{
    struct proc *curproc = myproc();

    if (addr >= curproc->sz || addr + 4 > curproc->sz) {
        return -1;
    }
    // ...
}
```

Now we can just cast the address to a pointer, dereference it, and store the
value in `*ip`.
```c
int fetchint(uint addr, int *ip)
{
    // ...
    *ip = *(int *)(addr);
    return 0;
}
```

Note that we can use an address like `addr` which will be in the lower half of
memory because traps don't perform a full context switch, so we're still using
the process's page directory even though we're in kernel mode (ring 0). If we
had switched to a kernel page directory, we'd have to call `walkpgdir()` or
`uva2ka()` to figure out the corresponding kernel virtual address for `addr`.

Now hopefully, if you've taken anything away from my past rants about undefined
behavior in C, you noticed something wrong with this function. If you didn't,
take another look; I'll wait.

Did you see it? We're dereferencing `addr` without checking that it's not null,
so if the user passed in a null address, we'd dereference a null pointer! We
also dereference `ip` without a similar check, but at the very least `ip` is
passed in by the kernel.

This could be very dangerous -- in general, it's undefined behavior in C, but
now that we've seen the code for handling traps, we're actually at a point where
we can figure out what would happen in xv6 if a null pointer gets dereferenced,
so let's take the opportunity to think about it for a bit.

First, what would happen if the kernel dereferenced a null pointer? Well, if the
kernel is currently using `kpgdir` as a page directory, the address 0 isn't
mapped to anything, so when the paging hardware goes to figure out which physical
address corresponds to the kernel virtual address 0, it would fail and generate
a "General Protection Fault" (trap number 13, or `T_GPFLT`). That would start
running the trap handler code, which would eventually get to the `switch`
statement in `trap()` (see the last post). Trap number 13 would fall under the
`default` case, and the if statement there would recognize that it originated in
the kernel. So it would print an error message to the console, then panic.

Okay, what if we're using a process's page directory, e.g. during a system call?
Address 0 is in the lower half of memory, so it's a user virtual address. The
result will depend on whether that page and its page table are mapped in the
process's page directory. If they are, then dereferencing a null pointer might
be fine after all. But if they're not mapped, dereferencing a null pointer will
cause a General Protection Fault. This time, `trap()` would print an error
message to the console, then mark the process to be killed.

Now, killing a process or causing a kernel panic might not sound like a huge
deal. In fact, xv6 does a great job here by killing a process that might have
dereferenced a null pointer or caused the kernel to do so. A kernel panic would
be much worse -- think about how annoying it would be if that PDF you downloaded
from that one sketchy website installed some malware that made your kernel panic
all the time -- the OS would become unusable. In fact, this is an example of a
"denial of service" vulnerability -- a malicious process might not be able to
read or write arbitrary memory or execute arbitrary code, but it can still keep
you from using your machine the way you expect to.

Just like `uva2ka()`, this function will only get called by one other function
(we'll see it soon), so it just so happens that under the current xv6 code,
it'll all be okay because it should never get passed a null pointer. But
everything from my rant about `uva2ka()` applies here: if you add any kernel
code that calls this function, be *VERY* careful and add your own null checks.

Okay, deep breath now. /rant.

### fetchstr

Fetching a string argument is tricky too; strings in C are just pointers to an
array of characters that ends in nul, i.e. `'\0'`, so this time we have to make
sure that both the pointer *and* the entire string are in the user's address
space; otherwise, we could unwittingly read from some arbitrary memory location
and pass the data back to the user process.

So we'll start by making sure the pointer itself is in a valid address:
```c
int fetchstr(uint addr, char **pp)
{
    struct proc *curproc = myproc();

    if (addr >= curproc->sz) {
        return -1;
    }
    // ...
}
```

Now we'll store the string pointer in `*pp`. We'll also get a pointer to the end
of the process's virtual address space so we can make sure the entire string is
inside its bounds.
```c
int fetchstr(uint addr, char **pp)
{
    // ...
    *pp = (char *) addr;
    char *ep = (char *) curproc->sz;
    // ...
}
```

How can we check if the entire string is inside user memory? Well, a string ends
with a nul byte, `'\0'`, so we just have to start scanning the memory starting
from `*pp` up to `ep` until we find a zero byte. If we find one in that range,
then the entire string is in user memory and we can return its length to
indicate success; otherwise the string overflows past the end of the process's
virtual address space, so we should return -1 to indicate failure.
```c
int fetchstr(uint addr, char **pp)
{
    // ...

    // Scan for a nul byte inside process's address space
    for (char *s = *pp; s < ep; s++) {
        if (*s == 0) {
            // If nul byte found, return the length
            return s - *pp;
        }
    }
    // String is not nul-terminated inside process's memory, so report failure
    return -1;
}
```

Note that again, we're dereferencing `pp` and `addr` without any null checks
(where `addr` is definitely the bigger concern, since it's user-generated), and
again, it's gonna work out okay (a misbehaving process will just get killed),
but once more: be careful if you use this function for your own kernel hacks.

### argint

This is the main function that the `sys_` system call functions will use to
recover an integer argument; it's basically just a wrapper for `fetchint()`. The
arguments are an integer `n` to say we want the nth integer argument, and a
pointer `ip` to store the recovered argument in. We have to call `fetchint()`
with an address argument, so the main task now is to figure out where in memory
the nth integer argument should be.

We're gonna have to use the x86 function call conventions again. Remember how
whenever we call a function in x86, its arguments get pushed onto the stack in
reverse order (i.e., from right to left), so that the first argument is at the
top of the stack (i.e., lowest memory address)? Then we push a return address
(`%eip`) and the old stack base pointer `%ebp`. Normally, the stack pointer
would just keep going on to the next slot on the callee's stack, but in this
case the code in `alltraps()` saved all the registers (including the stack
pointer `%esp`) in a `struct trapframe` before calling `trap()` or `syscall()`.

That means we can recover the old value of `%esp` from the trap frame and look
one spots below that on the stack (i.e., 4 bytes higher in memory, since `int`s
are 4 bytes) to get the first (`n = 0`) argument. The second argument (`n = 1`)
would be 8 bytes higher than `%esp`, and so on. Pretty neat.

Okay, now that we've got that down, the code for this function is pretty
straightforward.
```c
int argint(int n, int *ip)
{
    return fetchint((myproc()->tf->esp) + 4 + 4*n, ip);
}
```

### argptr

Some of the system call functions will have pointer arguments, so this function
recovers them. Pointers are 4 bytes in x86, so we can use `argint()` to get the
pointer itself before performing some additional checks to make sure the pointer
and the address it points to are valid.

The arguments are `n` (to retrieve the nth function argument), a pointer `pp` to
an address where we can store the retrieved pointer, and the size of the block
of memory that the retrieved pointer points to.

Let's start off by just retrieving the value of the pointer as an integer using
`argint()`; that'll make sure that the number `n` is valid.
```c
int argptr(int n, char **pp, int size)
{
    struct proc *curproc = myproc();

    int i;
    if (argint(n, &i) < 0) {
        return -1;
    }
    // ...
}
```

Now we have to make sure that the pointer we just retrieved is itself valid,
i.e. that the size is nonnegative and the beginning and end of the memory block
it points to are both within the process's address space.
```c
int argptr(int n, char **pp, int size)
{
    // ...
    if (size < 0 || (uint) i >= curproc->sz || (uint) i + size > curproc->sz) {
        return -1;
    }
    // ...
}
```

Finally, we can store the pointer in `*pp` and return 0.
```c
int argptr(int n, char **pp, int size)
{
    // ...
    *pp = (char *) i;
    return 0;
}
```

### argstr

A string is just a pointer in C, so we can recover the pointer's value using
`argint()` again, then pass it to `fetchstr()`. The former will make sure `n` is
valid, and the latter will make sure the string is nul-terminated and resides
entirely in the process's address space.
```c
int argstr(int n, char **pp)
{
    int addr;
    if (argint(n, &addr) < 0) {
        return -1;
    }
    return fetchstr(addr, pp);
}
```

## sysproc.c

So now we know how `syscall()` will route a system call trap to the right `sys_`
function, and we've seen how those functions can recover arguments from the
process's stack. Let's see some examples in action; most of these will be simple
wrapper functions.

### sys_fork

All the hard work here is gonna be done by `fork()`, which will create a new
child process by cloning the parent process's virtual address space. We don't
need any arguments for this, so we'll just call `fork()`.
```c
int sys_fork(void)
{
    return fork();
}
```

### sys_exit

`exit()` closes out a process, but it puts it in the `ZOMBIE` state so that the
parent process can call `wait()` to find out it's done running. `exit()` should
never return, so we'll add a return value here to make the compiler happy, but
it should never get executed.
```c
int sys_exit(void)
{
    exit();
    return 0;
}
```

### sys_wait

This system call is the parent process's counterpart to `exit()`; it'll do as
its name says and wait until the child process exits.
```c
int sys_wait(void)
{
    return wait();
}
```

### sys_kill

The `kill()` system call sounds like a more aggressive version of `exit()`:
after all, we're killing another process against its will, right? But in reality
it would be way too complicated to do that: the process might be running on
another CPU, midway through updating some kernel data structure, or about to
wake up another process that's asleep. Killing it by force might screw up a lot
of other things.

So instead `kill()` just tags it with the `killed` field in its `struct proc`;
eventually either the process will call `exit()` on its own, or it'll generate
another trap, at which point the code in `trap()` will call `exit()` on it.

`kill()` needs an integer argument: the process ID for the process we wish to
kill. So now we can see the payoff of writing those functions above.
```c
int sys_kill(void)
{
    int pid;
    if (argint(0, &pid) < 0) {
        return -1;
    }
    return kill(pid);
}
```

### sys_getpid

The `getpid()` system call is so simple that it doesn't even have another
function for this `sys_getpid()` to call. We'll just return the PID for the
current process.
```c
int sys_getpid(void)
{
    return myproc()->pid;
}
```

### sys_sbrk

If you're not familiar with system calls like `brk()` and `sbrk()` on Unix
systems, here's what they do: they grow or shrink the virtual address space of a
process. `brk()` sets its new size to a specific maximum address; `sbrk()` grows
or shrinks the process by a certain size in bytes and returns its old size.
They're mostly used to implement higher-level memory management functions like
`malloc()`. Heh, "high-level" probably isn't high on your mind when you think of
adjectives for `malloc()`, right? Anyway, xv6 only has `sbrk()`, so let's check
out its `sys_` wrapper function.

We'll need an integer argument (the number of bytes to grow or shrink by), so
let's grab that.
```c
int sys_sbrk(void)
{
    int n;
    if (argint(0, &n) < 0) {
        return -1;
    }
    // ...
}
```

Now we can use `growproc()` from our posts on paging to grow the process by `n`
bytes. But we want to return the old size, so we'll have to grab that before we
change it with the call to `growproc()`.
```c
int sys_sbrk(void)
{
    // ...
    int addr = myproc()->sz;

    if (growproc(n) < 0) {
        return -1;
    }

    return addr;
}
```

### sys_sleep

The `sleep()` function is pretty interesting; we'll get to the implementation
details later, but let's talk about the broad strokes now. You might be familiar
with the `sleep()` system call in Unix systems; you pass it an integer (usually
in milliseconds) and it puts your process to sleep (i.e., leaves it inactive or
not running) for that amount of time.

However, `sleep()` plays a dual role in xv6: the kernel will call `sleep()` for
processes that need to wait while something else happens, e.g. waiting for a
disk to read or write data. That way the processes don't end up idly spinning in
a loop or something and wasting valuable CPU time.

Implementing that is tricky; there's no way to know how long it would take for
whatever condition the process is waiting on to be satisfied, so it's not like
we can just stick in a random amount of time in the call to `sleep()` and hope
the condition is satisfied by then. So instead the `sleep()` function will just
"put a process to sleep" (read: make its state `SLEEPING` so it can't be run by
the scheduler) on a *channel*, which is just an arbitrary integer. Then later on
the kernel can wake up any processes sleeping on that channel. So for example,
the kernel can put a process waiting on the disk to sleep using a specific
channel that's assigned to the disk; then when the next disk interrupt occurs it
can wake up any processes that might be sleeping on the disk channel.

Okay so that's all well and good for the kernel's use of `sleep()`. But what
about the regular old `sleep()` system call? The argument is an integer that
represents the number of ticks to sleep for; how are we gonna turn that into a
channel to sleep on?

The answer is pretty neat (at least I think so): we'll set the channel to the
address of the `ticks` counter. Remember, `ticks` is a global variable that gets
incremented with every timer interrupt. Go check out the code in `trap()` again:
each timer interrupt sends a wakeup call to any processes that might be sleeping
on the `&ticks` channel. That should wake the process at every timer interrupt.
Then we'll just stick that inside a for loop so it keeps sleeping forever until
the right amount of ticks have passed.

Let's start by retrieving the integer argument, which is the number of ticks to
sleep for.
```c
int sys_sleep(void)
{
    int n;
    if (argint(0, &n) < 0) {
        return -1;
    }
    // ...
}
```

That argument `n` is a relative count, since a user process won't necessarily
know how many ticks have already gone by. So let's get the current tick count
before we put the process to sleep.
```c
int sys_sleep(void)
{
    // ...
    acquire(&tickslock);
    uint ticks0 = ticks;
    // ...
}
```

Now we just have to write that while loop I mentioned above to put the process
to sleep until `n` ticks have passed. Since we started counting at `ticks0`, the
condition should be satisfied when `ticks - ticks0 == n`.

Two more details: first, we'll add a check inside the while loop to see if the
current process has been tagged to be killed; if so, we'll just return -1 so we
can hasten the process's actual death by letting it run more code so the kernel
will call `exit()` on it at the next trap. Second, the function `sleep()` takes
another argument in addition to the channel: a lock. It'll release the lock for
us and reacquire it before waking up so that a sleeping process doesn't hog a
lock when it doesn't need it.
```c
int sys_sleep(void)
{
    // ...
    while (ticks - ticks0 < n) {
        if (myproc()->killed) {
            release(&tickslock);
            return -1;
        }
        sleep(&ticks, &tickslock);
    }
    release(&tickslock);
    return 0;
}
```

### sys_uptime

The `uptime()` system call just returns the amount of ticks that have passed
since the system started. This is another one that's so simple it doesn't need
another function, so we'll take care of it all here.

We just acquire the lock for `ticks`, get its current value, release the lock,
and return the value we got.
```c
int sys_uptime(void)
{
    acquire(&tickslock);
    uint xticks = ticks;
    release(&tickslock);
    return xticks;
}
```

## Running System Calls from User Code

We have system calls now! Well, not quite -- we still have to check out the
actual functions like `exit()`, `sleep()`, `kill()`, etc. Plus, we only saw the
`sys_` wrapper functions for *some* of the system calls here; the rest are in
[sysfile.c](https://github.com/mit-pdos/xv6-public/blob/master/sysfile.c), which
we'll get to after we understand the xv6 file system.

But let's pause for a second and think about how a user process will send a
system call. Like let's say you're writing some C code for a user program that
will run on xv6 and you want to create a child process with `fork()`. What
should you do?

Well, if you were coding for a Unix system like Linux or macOS, you'd just write
a call to `fork()` in your code. But that can't be right in xv6, can it? After
all, `fork()` is a kernel function, to be run in kernel mode with a current
privilege level of 0. Plus, isn't it supposed to be called by `sys_fork()`? So
should we call that?

None of these options will work. Well, yes, you do end up just calling `fork()`,
but it's *not* the kernel function `fork()`, so if you're expecting that one,
you'll be surprised when it doesn't behave the way you want it to. You won't be
able to use any kernel code at all in your user program for xv6. This is a
mistake I've seen a *lot* of people make in their xv6 OSTEP projects, so bear
with me for a second while I explain why you can't do it; feel free to skip the
next section on the Makefile if you already know why.

### Makefile

To see why, let's check out the xv6
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile) to see
how xv6 is actually compiled, built, and run. There's a ton of stuff in there,
but take a second to think about this: how do you usually run xv6? I bet it's
a command like `make qemu` or `make qemu-nox`, right?

If you're not familiar with Makefiles, here's a quick primer: each command like
`make qemu`, `make clean`, etc. is specified in the Makefile with a rule that
looks like this:
```make
mycmd: dependency1 dependency2 ...
    build_cmd1
    build_cmd2
    # ...
```
So if I run `make mycmd`, the `make` program will check that `dependency1`,
`dependency2`, etc. are up to date; if they're not, it'll update them by looking
up *their* rules and executing those to update them. Then it'll execute
`build_cmd1` on the shell, followed by `build_cmd2`, etc.

Okay, I know that might be confusing, so let me simply the `make qemu` command a
bit to make it more readable (note that I cut a lot of stuff out here, so don't
try to run xv6 with what I wrote below).
```make
# ...
qemu: fs.img xv6.img
    qemu -drive file=fs.img,index=1 -drive file=xv6.img,index=0
# ...
```
This just says that in order to run `make qemu` when you type it on the
terminal, the `make` program first has to make sure that both `fs.img` and
`xv6.img` are fully up to date. Then once they are, it can just run the shell
command `qemu` with the options `-drive file=fs.img,index=1` and
`-drive file=xv6.img,index=0`. Those options are just regular flags like the
ones you're probably used to with stuff like `ls -a` or `rm -rf`. In this case,
they tell `qemu` to use the files `fs.img` and `xv6.img` as virtual hard drives,
with `xv6.img` as disk number 0 and `fs.img` as disk number 1.

Okay, let's check out the `make` command for `xv6.img` next.
```make
# ...
xv6.img: bootblock kernel
    # some dd commands here
# ...
```
Hey, that's interesting, we already saw `bootblock` in a prior post. That's the
one we get when we compile the boot loader. `kernel` is, well, all the kernel
code. The `dd` command is often used in Unix systems to format and set up disks;
the details aren't important here, so I left them out for now. The point is that
the boot loader got compiled separately from the kernel code, remember? But
their machine code files get smushed into the same (virtual) disk together as
`xv6.img`, which will be disk 0 when we run in `qemu`.

Not let's check out the (slightly simplified) `make` command for `fs.img`.
```make
# ...
UPROGS = cat echo forktest grep init kill ln ls # ...

fs.img: mkfs README $(UPROGS)
    ./mkfs fs.img README $(UPROGS)
# ...
```
Okay, so `UPROGS` is just a list of all the user programs. Each of those gets
compiled separately; e.g. if you look in their source code, you'll see each one
has its own `main()` function. Then the shell command says to run `mkfs` to
create a file system called `fs.img` with `README` and all the user programs as
files.

The point of this detour is this: the boot loader gets compiled as a single
unit, as does the entire kernel code. But the user programs are compiled one at
a time. So if you write a user program for xv6, you should add it to the list in
`UPROGS` (as well as in `EXTRA`) and expect it to get compiled individually and
stuck onto the `fs.img` disk.

That means there's no way for a user program to call into any kernel code; the
linker wouldn't even be able to match up the call to the right function. So no
user program will ever be able to call functions like (the kernel's) `fork()`.
Think about it: if you write a program in C and compile it to run on Linux, do
you expect to have to recompile the entire Linux kernel just to run your one
little program? No, right?

But certainly we can't just expect every single program ever to be totally self-
contained. You also don't have to rewrite and recompile all of `malloc()` every
time you write a C program. So operating systems provide libraries for users to
include and call in their programs. Aha! So all we need to do in order for
user processes to execute system calls is to provide a library. That library is
[usys.S](https://github.com/mit-pdos/xv6-public/blob/master/usys.S).

### usys.S

Let's trace back to the beginning of a trap. In order to execute a system call,
we're supposed to send the processor an `int` instruction with a specific trap
number; that would be `int 64` for system calls on xv6. We're also supposed to
stick the system call number in the `%eax` register. Let's say we want to call
`fork()`. According to
[syscall.h](https://github.com/mit-pdos/xv6-public/blob/master/syscall.h), the
system call number for fork is `SYS_fork`, or 1. In order to send a specific x86
instruction and manipulate individual registers, we'll have to write our system
call library in assembly. Here's what it would look like for the `fork()` system
call:
```asm
.globl fork
fork:
    movl    $1, %eax
    int     $64
    ret
```

Okay, that's easy enough, but we have 21 of these to write, and it would be
pretty easy to make a mistake or write the wrong system call number. Let's
automate it instead with a C preprocessor macro. We've seen plenty of examples
of defining simple constants with `#define` directives for the preprocessor, but
we haven't looked at them too closely until now.

The C preprocessor is a piece of software that edits C (or assembly) code before
it's compiled. Preprocessor directives like `#define A 5` create macros that are
expanded to replace every instance of `A` in the code with the number 5;
directives like `#include "header.h"` expand such that they essentially copy-
paste all the code in the file `header.h`. We can also create function-like
macros like the `P2V()` and `V2P()` macros we've used often by adding a
parameter inside parentheses; unlike functions, these will be expanded *before*
compilation to paste the code into every instance of its use, thus avoiding the
usual overhead associated with a function call. Function-like macros are also
generic, in a sense, since they don't require specifying parameter types or
return types (as long as it works within the places where the macro will be
used). Note that there are some drawbacks: macros aren't type-checked, they can
evaluate their arguments more than once, we can't use pointers to them like we
can with functions, and they can result in larger code.

We're gonna use a function-like macro here to create the assembly code for each
system call function so that it gets expanded before the code is assembled.
We'll use `T_SYSCALL` instead of 64 in the code above, and `SYS_fork` (or its
equivalent for each system call) for the system call number. We'll have to
replace the part after the underscore in `SYS_` with the name of the system call
function; we can do that with the token-pasting operator `##`, which glues two
tokens together to form a single token. Also, macros must be defined on a single
line, so we'll escape the newline characters with `\` and end each assembly line
with a semicolon.
```asm
#include "syscall.h"
#include "traps.h"

#define SYSCALL(name) \
    .globl name; \
    name: \
        movl    $SYS_##name, %eax; \
        int $T_SYSCALL; \
        ret
```

Now we can just invoke the macro on the name of each function we want to create:
```asm
# ...
SYSCALL(fork)
SYSCALL(exit)
SYSCALL(wait)
# and so on ...
```

After the preprocessor runs on the file, the result will look like this:
```asm
.globl fork
fork:
    movl    $1, %eax
    int     $64
    ret

.globl exit
exit:
    movl    $2, %eax
    int     $64
    ret

.globl wait
wait:
    movl    $3, %eax
    int     $64
    ret

# and so on...
```

Great! Now we have 21 functions for the system calls, all written in assembly.
All user programs for xv6 will be compiled together with the code for these
functions: see `ULIB` in the Makefile. So now, a user program can execute a
system call by calling these functions, e.g. `fork()`.

## Summary

After all the preparations are handled by the trap handler functions in the IDT,
`alltraps()`, and `trap()`, system calls get routed to the `syscall()` function,
which uses a system call number to pick the right function out of an array. That
function will have to recover any arguments to the system call before passing it
on to the real system call function later on.

Next up, we'll take a look at some of those system calls; we'll leave the rest
until after we go over xv6's file system.
