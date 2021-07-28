# Devices: Disk Driver

At this point, we've seen how xv6 virtualizes memory and the processor to give
each user process the illusion of a contiguous, near-infinite memory space and a
dedicated CPU to run it; we've also seen how xv6 mediates interactions between
most of a computer's hardware components and user processes via system calls.
But there's one more piece of hardware that's critically important for an OS
that we haven't looked at yet: the disk. All that's left in the kernel code for
us to look at is how xv6 manages data storage on the disk and how it presents
that data to users in a simplified way.

The function of a disk is to provide *persistence* for an operating system. RAM
is volatile memory: it gets erased when the machine is turned off, so any data
stored there is fleeting. A disk allows an OS to store and retrieve data across
shut-offs. The disk driver we'll go over in this post allows the xv6 kernel
direct access to that device so it can read and write data to it.

But unlike other devices, a simple driver isn't enough here. We don't just need
to be able to read and write data; we'd like to present users with a simplified,
accessible framework to navigate that data. Imagine using a computer where you
had to specify which byte of the disk to read or write, then remember that
yourself in order to access it again later. It's madness! Enter file systems;
"files" don't really exist in any real sense on a disk, but the OS can provide
the illusion of discrete, individual files in order to simplify access to data.

We also need to make sure concurrent accesses of the same file don't risk
corrupting the file (or even the entire file system). We need to separate out
kernel data (like the kernel code itself) from user data on the disk, so that a
malicious user process can't just overwrite arbitrary kernel code. Finally,
there's that oh-so-famous line about Unix systems, "everything is a file". We'll
need a way to present "everything" in the elegant abstraction of a file.

All of these abstractions and security checks will require far more code than a
simple driver to implement them, so before we go on to the driver, let's check
out how xv6 will organize its file system to get a preview of what's ahead.

## File System Organization

Laying the abstraction of a complete file system on top of a physical disk will
require several steps. xv6 does this using seven layers. From bottom (direct
hardware interaction) to top (user-facing code), they are:
* Disk driver: reads and writes blocks on an IDE hard drive.
* Buffer cache: caches disk blocks in memory and synchronizes access to them.
* Logging: provides atomic disk writes to mitigate the risk of a crash.
* Inodes: turns disk blocks into individual files that the OS can manipulate.
* Directories: creates a tree of named directories that contain other files.
* Path names: provides hierarchical, human-readable path names in the directory tree structure.
* File descriptors: abstracts OS resources like pipes and devices as files to provide a unified API for user programs.

That's a lot of work to do now, but it'll pay off! The kernel will do all this
labor so that users are free to be lazy later on and can live in blissful
ignorance of the fact that their precious little files actually exist as nothing
but ones and zeroes in totally arbitrary locations on the disk.

Note that hard drives are usually divided into *sectors*, which are physical
divisions (originally referring to literal geometric sectors), traditionally of
512 bytes. Operating systems can then collect these into larger *blocks* which
are multiples of the sector size. xv6 uses 512-byte blocks for simplicity so
that the sector and block sizes match up; I'll use the two terms interchangeably.

On the disk, block 0 usually contains the boot sector, so it's not used by xv6
(but remember the Makefile -- xv6 actually stores the boot loader and kernel
code on an entirely separate physical disk). Block 1 is called the *superblock*
because it contains metadata about the file system like its total size, the size
of the log, the number of files, and their location on the disk. Then the log
starts at block 2 and on.

## buf.h

If you've read any of the previous optional posts on device drivers, you know
that interacting directly with the hardware means all kinds of opaque code with
seemingly-arbitrary port I/O and cryptic magic numbers. Drivers are also specific
to the actual (or virtual) hardware in the machine that xv6 will run on, so it
tends to be less useful for showing general OS concepts -- hence why all the
other device driver posts were optional. That being said, the disk driver nicely
rounds out the rest of the file system code, so I recommend checking it out, but
if you're short on time or bored with all the talk about hardware specs, feel
free to skip to the summary section below.

Reading and writing disk data is super slow, so the second layer in the file
system is the buffer cache, which will store copies of disk blocks in memory for
faster access. But we still have to read from the disk to create that buffer,
and we still have to write any modified data to the disk once we're done, so
we still need a layer below the buffer cache to do that. That layer is the disk
driver; its purpose is to copy data from the disk to the in-memory cache and
vice versa. A single block is represented in the cache as a `struct buf`, defined
in [buf.h](https://github.com/mit-pdos/xv6-public/blob/master/buf.h).
```c
struct buf {
    int flags;
    uint dev;               // device number
    uint blockno;           // block number (same as sector number)
    struct sleeplock lock;  // sleep-lock to protect buffer reads and writes
    uint refcnt;            // how many processes are using this buffer
    struct buf *prev;       // for use with buffer cache doubly-linked list
    struct buf *next;       // for use with buffer cache doubly-linked list
    struct buf *qnext;      // for use with disk driver queue
    uchar data[BSIZE];      // data stored in the buffer
};

#define B_VALID 0x2
#define B_DIRTY 0x4
```

The two constants defined at the bottom are used in the `flags` field; `B_VALID`
indicates that a buffer has been read from disk and should accurately reflect
the sector's contents on the disk, and `B_DIRTY` says we've modified the buffer
but haven't yet updated the on-disk version of a file, so we need to write the
buffer to disk soon.

We'll see later on that the buffer cache uses a doubly-linked list of buffers;
the `prev` and `next` fields are used there. However, the disk driver also
maintains its own queue of buffers that are waiting to be read from or written
to the disk; that's implemented as a singly-linked list using the `qnext` field.

## ide.c

We've already seen some code to read and write disk data in the [boot loader](boot.md);
I know it's been a while, so you can check that out again if you want. We can't
reuse the code there for a few reasons, though: (1) the boot loader has to be
compiled separately from the kernel, so we can't access any of the functions
there, and (2) we need to store data in the buffer cache, so we can't even copy-
paste the code we used before since the boot loader barely even knows what
memory is, let alone a buffer cache.

### ATA Programmed I/O Mode

Modern disk drivers usually talk to the disk via direct memory access (DMA), but
to keep things simple xv6 is just gonna talk to it with port I/O. That's much,
much slower, and it requires active participation by the CPU (which means it
can't do anything else at the same time), but hey, xv6 thinks it's 1995,
remember? So PIO mode is still (relatively) cutting edge. Either way, extreme
performance isn't the goal here, so we'll just have to suck it up.

Okay, let's do a super-quick summary. `inb` is a C wrapper for an x86 assembly
instruction that reads a single byte of data from a port; `outb` writes a byte
to a port. The disk controller chip has primary and secondary buses; the primary
bus sends data on port 0x1F0 and has control registers on ports 0x1F1 through
0x1F7. Port 0x1F7 doubles as a command register and a status port with some
useful flags we can check in order to know what the disk is up to; we saw some
of those before, but I'll give you the full list now.
* Bit 0 (0x01) - ERR (indicates an error occurred)
* Bit 1 (0x02) - IDX (index; always set to zero)
* Bit 2 (0x04) - CORR (corrected data; always set to zero)
* Bit 3 (0x08) - DRQ (drive has data to transfer or is ready to receive data)
* Bit 4 (0x10) - SRV (service request)
* Bit 5 (0x20) - DF (drive fault error)
* Bit 6 (0x40) - RDY (ready; clear when drive isn't running or after an error and set otherwise)
* Bit 7 (0x80) - BSY (busy; drive is in the middle of sending/receiving data)

The disk driver defines some of these with preprocessor macros at the top of the
file.
```c
#define SECTOR_SIZE 512
#define IDE_BSY     0x80
#define IDE_DRDY    0x40
#define IDE_DF      0x20
#define IDE_ERR     0x01
// ...
```

We also saw one command example in the boot loader: sending 0x20 to port 0x1F7
tells the disk to read a sector and send it to us through data port 0x1F0. Now
we'll also use commands to write a sector, as well as to read or write multiple
sectors at once.
```c
// ...
#define IDE_CMD_READ    0x20
#define IDE_CMD_WRITE   0x30
#define IDE_CMD_RDMUL   0xc4
#define IDE_CMD_WRMUL   0xc5
// ...
```

If, for some reason beyond mortal comprehension, you decide you want to know
more about the eldritch secrets of ancient hard drives, you can read [this
resource on ATA disks](https://pdos.csail.mit.edu/6.828/2018/readings/hardware/ATA-d1410r3a.pdf).

After those constants, we find three static global variables: a spin-lock for
accessing the disk, the queue of buffers waiting to be synchronized with their
on-disk counterparts, and a boolean to track whether xv6 is running with only
disk 0 (boot loader and kernel) or with disk 1 (user file system) as well.
```c
// ...
static struct spinlock idelock;
static struct buf *idequeue;
static int havedisk1;
// ...
```

### idewait

This function takes an integer `checkerr` argument that should be a boolean and
waits for the disk to be ready to receive more commands. If `checkerr` is true,
it'll also check whether the status port includes any error flags.

It starts by reading from the disk's status port and looping until the busy
flag is not set but the ready flag is. The bitwise-OR `IDE_BSY | IDE_DRDY`
combines both flags, and the bitwise-AND tests whether either one is set in `r`.
```c
static int idewait(int checkerr)
{
    int r;
    while (((r = inb(0x1f7)) & (IDE_BSY | IDE_DRDY)) != IDE_DRDY)
        ;
    // ...
}
```

Now if `checkerr` is nonzero we have to check that neither the error nor the
drive failure flag is set in the status port. If either one is set, we'll return
-1; we'll return 0 otherwise.
```c
static int idewait(int checkerr)
{
    // ...
    if (checkerr && (r & (IDE_DF | IDE_ERR)) != 0) {
        return -1;
    }
    return 0;
}
```

### ideinit

This function is called by the kernel's `main()` during set-up to initialize the
disk. We start by initializing the disk lock, then tell the I/O interrupt
controller to forward all disk interrupts to the last CPU. We talked about the
`ioapicenable()` function in detail in the post on interrupt controllers.
```c
void ideinit(void)
{
    initlock(&idelock, "ide");
    ioapicenable(IRQ_IDE, ncpu - 1);
    // ...
}
```

Then we wait for the disk to be ready to accept commands (ignoring any error
flags that may be present).
```c
void ideinit(void)
{
    // ...
    idewait(0);
    // ...
}
```

We said above that disk 0 should contain the boot loader and kernel, so we can
assume any machine running xv6 should have that present. However, we need to
make sure disk 1 is present; the
[Makefile](https://github.com/mit-pdos/xv6-public/blob/master/Makefile) includes
some configurations like `make qemu-memfs` under which xv6 can run without a
dedicated disk for the file system, storing files in memory instead.

Port 0x1F6 is used to select a drive. Bits 5 and 7 should always be set, and bit
6 picks the right mode we need to indicate a disk. Bit 4 determines whether we
want to select disk 0 or disk 1. So we can select drive 1 by setting bits 5-7
(0xE0 when combined), then bit 4 (`1 << 4`).
```c
void ideinit(void)
{
    // ...
    outb(0x1f6, 0xe0 | (1 << 4));
    // ...
}
```

Now we need to wait for disk 1 to be ready; we need to handle this as a special
case since `waitdisk()` can't check a specific disk for us, and because an
absent disk 1 would make the while loop there continue forever. So we'll check
the status register 1000 times; if it ever reports that it's ready, we'll set
`havedisk1` to true and break, but otherwise we'll assume disk 1 isn't present
and leave `havedisk1` as zero (i.e., false).
```c
void ideinit(void)
{
    // ...
    for (int i = 0; i < 1000; i++) {
        if (inb(0x1f7) != 0) {
            havedisk1 = 1;
            break;
        }
    }
    // ...
}
```

Finally, we'll switch back to using disk 0 by changing the fourth bit of the
register at port 0x1F6.
```c
void ideinit(void)
{
    // ...
    outb(0x1f6, 0xe0 | (0 << 4));
}
```

### idestart

This is the core function that will read or write a buffer to or from the disk.
It's a `static` function, so it can only be called by other functions in this
file; `ideintr()` and `iderw()` will both use it as a helper function. It takes
a pointer to a buffer, so the first thing to do is make sure that pointer isn't
null. We'll also make sure the buffer's block number is within the maximum limit
set by `FSSIZE`, defined in
[param.h](https://github.com/mit-pdos/xv6-public/blob/master/param.h) as 1000.
```c
static void idestart(struct buf *b)
{
    if (b == 0) {
        panic("idestart");
    }
    if (b->blockno >= FSSIZE) {
        panic("incorrect blockno");
    }
    // ...
}
```

Next we need to figure out which disk sector to read from or write to. Since xv6
uses blocks that are the same size as a sector, this should just be `b->blockno`,
but we'll add a conversion here in case that gets changed later on (especially
if we want higher disk throughput).
```c
static void idestart(struct buf *b)
{
    // ...
    int sector_per_block = BSIZE / SECTOR_SIZE;
    int sector = b->blockno * sector_per_block;
    // ...
}
```

If each block fits exactly one sector, then we'll need to use the single-sector
read and write commands; otherwise we should use the multi-sector versions of
those commands. We'll set `read_cmd` and `write_cmd` to the right versions.
We'll also make sure that there are no more than 7 sectors per block.
```c
static void idestart(struct buf *b)
{
    // ...
    int read_cmd = (sector_per_block == 1) ? IDE_CMD_READ : IDE_CMD_RDMUL;
    int write_cmd = (sector_per_block == 1) ? IDE_CMD_WRITE : IDE_CMD_WRMUL;
    if (sector_per_block > 7) {
        panic("idestart");
    }
    // ...
}
```

Now let's wait for the disk to be ready, ignoring any error flags.
```c
static void idestart(struct buf *b)
{
    // ...
    idewait(0);
    // ...
}
```

Okay, now it's time to brace yourself, because this next part is a hot mess of
port I/O operations with lots of magic numbers. First we'll tell the disk
controller to generate an interrupt once it's done reading or writing by setting
the device control register at 0x3F6 to zero. Then we'll tell it how many total
sectors we want to read or write by writing that number (AKA `sector_per_block`)
to port 0x1F2.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x3f6, 0);                 // generate interrupt when done
    outb(0x1f2, sector_per_block);  // number of sectors to read/write
    // ...
}
```

Before sending the read or write command, we have to tell the disk which sector
to read from, using our `sector` variable from above. Let's take a second to
talk about hard drive geometry. A hard drive consists of a bunch of stacked
circular surfaces, where each surface has a corresponding *head* that changes
its position to read or write from the right place on the disk. Each surface has
a number of *tracks*: concentric circles that contain data. If you pick a track
number (i.e. pick a distance from the center of the surfaces) and collect all
those tracks from all the surfaces, you get a *cylinder*.

A sector number acts as a kind of address with each part specifying a different
geometric component, similar to how linear addresses contain a page directory
index, page table index, and offset. The eight most significant bits (24 through
31) identify the drive and/or head that the sector is located on (plus some
flags); bits 8 through 23 identify the cylinder, and bits 0 through 7 pick a
sector within that cylinder. Altogether, these define a 3D coordinate system
that uniquely identifies all sectors on a machine's disks.

Port 0x1F3 is the sector number register, ports 0x1F4 and 0x1F5 are the cylinder
low and high registers, and port 0x1F6 is the drive/head register. We can write
the sector number as `sector & 0xFF`; the cylinder low and high numbers can be
recovered by bitshifting `sector` down by 8 and 16, respectively.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x1f3, sector & 0xff);             // sector number
    outb(0x1f4, (sector >> 8) & 0xff);      // cylinder low
    outb(0x1f5, (sector >> 16) & 0xff);     // cylinder high
    // ...
}
```

Now for the drive/head register, we'll use `b->dev` to get the block's device
and `(sector >> 24)` to get the head it's on. Finally, we'll set bits 5-7 as
required (and as mentioned above in `ideinit()`) with 0xE0. Then we can
bitwise-OR all of these together and write them to port 0x1F6.
```c
static void idestart(struct buf *b)
{
    // ...
    outb(0x1f6, 0xe0 | ((b->dev & 1) << 4) | ((sector >> 24) & 0x0f));
    // ...
}
```

Okay, that was the worst of it! Deep breath now. The last part is just sending
the actual read or write command. But how do we know which one we're supposed to
do? The only argument is a pointer to a buffer `b`, not any sort of boolean that
might tell us which to carry out. Well, remember the buffer flag `B_DIRTY`? That
one indicates that a buffer has been modified and needs to be written to disk.
If that flag is set, reading from the disk would overwrite any changes, which
probably isn't what we want. So let's just assume that the `B_DIRTY` flag means
we should write to disk, and the absence of that flag means we should read from
disk.
```c
static void idestart(struct buf *b)
{
    // ...
    if (b->flags & B_DIRTY) {
        outb(0x1f7, write_cmd);
        outsl(0x1f0, b->data, BSIZE / 4);
    } else {
        outb(0x1f7, read_cmd);
    }
}
```
Here `outsl()` is another C wrapper for an x86 instruction; this one writes data
from a string, four bytes at a time.

That's it! This is by far the most cryptic function in the disk driver; the last
two are relatively easy now.

### ideintr

We saw in `idestart()` that we set up the disk to send an interrupt whenever
it's done reading or writing data. Back when we looked at
[trap.c](https://github.com/mit-pdos/xv6-public/blob/master/trap.c), we saw that
the `trap()` function directs all disk interrupts to the handler function
`ideintr()`. It's time to check that one out now.

We'll start by acquiring the disk's spin-lock; note that we don't use a sleep-
lock because this is an interrupt handler function, so interrupts should be
disabled while it runs.
```c
void ideintr(void)
{
    acquire(&idelock);
    // ...
    release(&idelock);
}
```

If we got an interrupt, then it usually means the disk is done with the most
recent request. Those requests are stored in the global `idequeue` linked list,
with the current request at the front of the queue. So we'll get the head of the
queue as `b`, then set `idequeue` to point to the next buffer in the queue. If
the head is null, then we'll just return early.
```c
void ideintr(void)
{
    // ...
    struct buf *b;
    if ((b = idequeue) == 0) {
        release(&idelock);
        return;
    }
    idequeue = b->next;
    // ...
}
```

The read command in `idestart()` didn't specify where to read the data to, so we
do that now. We'll check if the `B_DIRTY` flag was set; if it wasn't (i.e. the
operation was a disk read), then we'll wait for the disk to be ready (without
any errors, using `idewait(1)` instead of `idewait(0)` as we have before) and
read the data into `b->data`.
```c
void ideintr(void)
{
    // ...
    if (!(b->flags & B_DIRTY) && idewait(1) >= 0) {
        insl(0x1f0, b->data, BSIZE / 4);
    }
    // ...
}
```

Next, we set the `B_VALID` flag with a bitwise-OR and clear any `B_DIRTY` flag
with a bitwise-AND and a bitwise-NOT. Then we'll wake up any user process that
went to sleep on a channel for this buffer after requesting a disk I/O operation.
```c
void ideintr(void)
{
    // ...
    b->flags |= B_VALID;
    b->flags &= ~B_DIRTY;
    wakeup(b);
    // ...
}
```

Finally, we'll get the disk started on the next operation, for the next buffer
in the queue.
```c
void ideintr(void)
{
    // ...
    if (idequeue != 0) {
        idestart(idequeue);
    }
    // ...
}
```

### iderw

The `idestart()` function is `static`, so it can't be called by anything outside
of this file; we need to provide a mechanism for both kernel and user threads to
read and write disk data. That's what `iderw()` does. Note that processes should
never call this function directly; it only gets called by the code for the
buffer cache layer of the file system. In other words, processes will use system
calls like `open()`, `read()`, `write()`, `close()`, etc., which in turn will
use functions from higher layers of abstraction, which in turn call functions
from lower layers, and so on, until they reach the buffer cache, which calls
`iderw()` to finally read/write directly from/to the disk.

By the time a process gets to `iderw()`, it should already be holding a sleep-
lock `b->lock` for the buffer `b` it wants to read or write, and either the
`B_DIRTY` flag should be set (to write to disk) or the `B_VALID` flag should be
absent (to read from disk). We'll start off with some sanity checks for those,
and make sure that we're not trying to read from disk 1 if it's not present on
this machine. Then we'll acquire the disk's spin-lock.
```c
void iderw(struct buf *b)
{
    if (!holdingsleep(&b->lock)) {
        panic("iderw: buf not locked");
    }
    if ((b->flags & (B_VALID | B_DIRTY)) == B_VALID) {
        // B_VALID is set, so we don't need to read it; B_DIRTY is not set, so
        // we don't need to write it
        panic("iderw: nothing to do");
    }
    if (b->dev != 0 && !havedisk1) {
        panic("iderw: ide disk 1 not present");
    }

    acquire(&idelock);
    // ...
    release(&idelock);
}
```

There may be other buffers waiting in line in the disk queue, so we have to
append this buffer `b` to the end of `idequeue`. We can do that by setting
`b->qnext` to null, then creating a variable `pp` to traverse the entire queue.
When `pp` points to the last element, we'll set its `qnext` field to point to
`b`.
```c
void iderw(struct buf *b)
{
    // ...
    b->qnext = 0;

    // Traverse the queue
    struct buf **pp;
    for (pp = &idequeue; *pp; pp = &(*pp)->qnext)
        ;

    // Append b to end of queue
    *pp = b;

    // ...
}
```
That traversal might look confusing as all hell, so let's take a closer look.
It defines `pp` as a double pointer: a pointer to a pointer to a `struct buf`.
(If you've seen the interview of Linus Torvalds where he talks about good style
with linked lists, it's similar to the code there; there's a nice summary
[here](https://github.com/mkirchner/linked-list-good-taste).) `pp` starts off
equal pointing to `idequeue`, i.e. the head of the linked list. Each iteration
checks that `pp` points to a valid (non-null) pointer, i.e. the loop will end
when we reach the end of the list. The body of the loop is empty, so none of the
iterations actually do anything; the purpose of the for loop is just to update
`pp` several times. At the end of each iteration, `pp` is updated to point to a
pointer to the next buffer in the queue.

Suppose the last buffer in the queue is `end`. At the end of the for loop, `pp`
will hold the address of `end->qnext`, so `*pp = b` sets `end->qnext = b`. The
double indirection makes it easy to update the last buffer in the queue; without
it, we would have to stop the loop one step earlier when `pp` points to `end`
instead of `end->qnext` then be careful to update the actual buffer at the end
of the queue instead of just updating the local variable `pp`. All in all, it's
just an elegant way to write a linked list traversal in a single line.

Okay, so now our buffer `b` is at the end of the queue. If there are others in
front of it, then `ideintr()` will make sure that each disk interrupt starts the
disk on the next operation. But what if `b` is actually the only buffer in the
queue? In that case, the disk isn't running yet, so we need to get it started
ourselves.
```c
void iderw(struct buf *b)
{
    // ...
    if (idequeue == b) {
        idestart(b);
    }
    // ...
}
```

At this point, we can be confident that the disk will either start our request
now or get to it eventually (if there are other requests in the queue). This
process just has to wait for the disk to finish, so we'll put it to sleep until
the buffer has been synchronized with the disk. We'll check that by making sure
the `B_VALID` flag is present but `B_DIRTY` is not set. The call to `sleep()`
will release `idelock` and reacquire it before returning.
```c
void iderw(struct buf *b)
{
    // ...
    while ((b->flags & (B_VALID | B_DIRTY)) != B_VALID) {
        sleep(b, &idelock);
    }
    // ...
}
```

## Summary

The disk driver handles direct communication with the hard drive, issuing orders
to read or write sectors. It exposes two API functions, `ideintr()` and
`iderw()`. The former is called by `trap()` to handle disk interrupts, while the
latter is called by the code for the buffer cache layer of the file system to
update blocks in the buffer cache with their corresponding sectors on disk. Next
up we'll look at the buffer cache itself, as well as the logging layer, which
provides crash recovery.
