# xv6-annotated

A detailed, line-by-line guide to the [xv6 kernel code](https://github.com/mit-pdos/xv6-public).

## Overview

I started taking notes on xv6 when I first read through the source code for the
projects in
[Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/).
Later on, I saw a lot of students struggling with xv6 and the
[OSTEP projects](https://github.com/remzi-arpacidusseau/ostep-projects/);
it seems like the biggest challenges are (1) the C language, especially when it
comes to low-level systems programming and compiling for a bare-metal target,
(2) finding a suitable entry point and a linear path to read through the code in
one go, and (3) the lack of a single, centralized guide to the source code.

[The book that accompanies xv6](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)
is great, but it doesn't completely solve those issues; it goes over the main
points, but I still found myself reading a lot of hardware specs,
[OSDev Wiki](https://wiki.osdev.org/) pages, and [*Computer Systems: A Programmer's
Perspective*](https://csapp.cs.cmu.edu/3e/home.html) to get the finer details. I
also felt like the xv6 book left out some of the juicy C nuggets that I knew
would be a challenge for students who haven't had to suffer through months of C
programming before coming to xv6.

Finally, there's a particular challenge that comes with the xv6 projects in
OSTEP: reading the xv6 source code requires familiarity with virtual memory and
locks right from the start, but neither the book nor the course get to those
until after the first xv6 project.

I absolutely loved reading through the source code and learned a ton about Unix
systems, so I hope you do too!

Please freel free to point out errors, corrections, improvements, etc.

## Contents

1. [Das Boot](boot.md)
2. [Entering the Kernel](entry.md)
3. [Spin-Locks](spin_locks.md)
4. [Paging: Page Allocation](page_allocation.md)
5. [Paging: Kernel Page Directory](paging_kernel.md)
6. [Paging: User Space and Processes](paging_user.md)
7. Devices: Multiprocessing (optional)
8. Devices: Interrupt Controllers (optional)
9. [Processes](processes.md)
10. [Scheduling](scheduling.md)
11. [It's a Trap!](traps.md)
12. [System Calls: Routing](syscalls_routing.md)
13. [System Calls: Processes](syscalls_processes.md)
14. Devices: Serial Port and Console Drivers (optional)
15. Devices: Keyboard Driver (optional)
16. [Sleep-Locks](sleep_locks.md)
17. [Devices: Disk Driver](disk.md)
18. File System: Lower Layers
19. File System: Upper Layers
20. System Calls: Files
21. User Space: The First Process
22. User Space: The Shell
23. User Space: C Library (optional)
24. User Space: Programs (optional)

## Roadmap

I recommend reading the posts in order; I'll assume you've read all the previous
posts in any later ones. You can feel free to skip any sections marked as
optional though. You should have a copy of the xv6 source code in front of you
while you read the posts. If you want a shorter read, you can read the xv6 book
instead; it's a great book, but it doesn't exhaustively analyze every single
line of code and it assumes more knowledge of C and x86 architecture than what
most [OSSU](https://github.com/ossu/computer-science/) students will have by the
time they get to look at xv6.

### Assumed Background

* I'll assume you've read through all the [OSTEP chapters on virtualization](https://pages.cs.wisc.edu/~remzi/OSTEP/#book-chapters) (or watched all the [lectures](https://pages.cs.wisc.edu/~remzi/Classes/537/Spring2018/Discussion/videos.html)) before starting this guide. That doesn't mean doing all the virtualization projects: you'll have to understand a lot about xv6 right off the bat, so hold off on the xv6 projects until after you've read this guide.
* You should be reasonably familiar with C, maybe at the level of something like CS 50 (but see the next bullet point). Please please please don't try to learn C from online resources like GeeksforGeeks, TutorialsPoint, or any of the pages listed on hackr.io -- I've seen too many examples of dangerous C practices or straight-up incorrect explanations to trust those websites at all. They might be great for other languages, but unfortunately C isn't the kind of language you can pick up from online tutorials; it has way too many pitfalls. Use a book like [*C Programming: A Modern Approach*](http://www.knking.com/books/c2/), [*Modern C*](https://modernc.gforge.inria.fr), [*The C Programming Language*](https://en.wikipedia.org/wiki/The_C_Programming_Language), or [*Effective C*](https://nostarch.com/Effective_C).
* However, I'm also gonna assume you're *not* an expert on C, so I'll go over some of the more obscure C features used in the xv6 code. If you're the kind of person who uses `#pragma`s in their code or can cite line numbers in the C Standard, then you can probably just read the xv6 code directly; this guide will just slow you down.
* You should take an introductory systems course like [Nand2Tetris](https://www.nand2tetris.org/) or [CS:APP](https://www.cs.cmu.edu/afs/cs.cmu.edu/academic/class/15213-f15/www/schedule.html) [(book)](https://csapp.cs.cmu.edu/3e/home.html) in order to understand the basics of computer architecture and systems programming. You don't need to know any x86-specific details or assembly language -- I'll talk about those along the way -- but it wouldn't hurt either.

### OSTEP Projects

If you're working through OSTEP, you should start off by doing the
[`initial-utilities`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/initial-utilities)
project. You can even do it *before* watching any lectures or reading any
chapters. It doesn't require any knowledge about operating systems, and it's a
good litmus test of your experience with C. An optional second project to do at
this point is [`initial-reverse`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/initial-reverse).

Then watch all the virtualization lectures or read chapters 3 through 24 in the
book. Do the homework assignments as you come across them; those are usually
simple scripts or (relatively) short coding exercises to check your understanding.
You can do the [`processes-shell`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/processes-shell)
project after chapter 5 of the book or discussion 3 of the 2018 OSTEP course. My
suggestion is to skip any other projects until you've read this guide -- the xv6
projects require a lot of detailed knowledge about its structure, even if you're
only gonna end up adding 10 lines of code for any given project.

Once you're done with virtualization in OSTEP, come back here and read these
posts, starting from the boot process up to the end of processes and system
calls. Then do
[`initial-xv6`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/initial-xv6),
[`scheduling-xv6-lottery`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/scheduling-xv6-lottery),
and [`vm-xv6-intro`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/vm-xv6-intro).

Then it's back to OSTEP for the concurrency chapters, after which you can do
[`concurrency-xv6-threads`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/initial-xv6)
(as well as the other concurrency projects that don't use xv6). Then do the
entire file system section of OSTEP, and read the rest of these posts. You can
wrap up xv6 with the [`filesystems-checker`](https://github.com/remzi-arpacidusseau/ostep-projects/tree/master/filesystems-checker)
project.


## Acknowledgements

Huge thanks to [@spamegg](https://github.com/spamegg1) from
[OSSU](https://github.com/ossu/computer-science/) for the many hours spent
reading the source code with me and helping me wrangle the OSTEP projects and
improve their documentation and test scripts to work for other OSSU students!

### xv6 License

The xv6 software is:

Copyright (c) 2006-2018 Franz Kaashoek, Robert Morris, Russ Cox,
						Massachusetts Institute of Technology

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES, OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

