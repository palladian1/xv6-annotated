# xv6-annotated

A detailed, line-by-line guide to the [xv6 kernel code](https://github.com/mit-pdos/xv6-public).

## Overview

I started taking notes on xv6 when I first read through the source code and the
book for the projects in [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/). Later on, I saw a
lot of students struggling with xv6 and the [OSTEP projects](https://github.com/remzi-arpacidusseau/ostep-projects/); it seems like the
biggest challenges are (1) the C language, especially when it comes to coding
for a bare-metal target, (2) finding a suitable entry point and a linear path to
read through the code in one go, and (3) the lack of a single, centralized guide
to the source code.

[The book that accompanies xv6](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf) is great, but it doesn't completely solve those
issues; it goes over the main points, but I still found myself reading a lot of
hardware specs, [OSDev Wiki](https://wiki.osdev.org/) pages, and the book [*Computer Systems: A Programmer's
Perspective*](https://csapp.cs.cmu.edu/3e/home.html) to get the finer details. I also felt like the xv6 book left out
some of the juicy C nuggets that I knew would be a challenge for students who
haven't had to suffer through months of C programming.

Finally, there's a particular challenge that comes with the xv6 projects in
OSTEP: reading the xv6 source code requires familiarity with virtual memory and
locks right from the start, but neither the book nor the course get to those
until after the first xv6 project.

I made a few assumptions in writing this:
* You've read through all the [OSTEP chapters on virtualization](https://pages.cs.wisc.edu/~remzi/OSTEP/#book-chapters) (or watched all the [lectures](https://pages.cs.wisc.edu/~remzi/Classes/537/Spring2018/Discussion/videos.html)). Hold off on the first project until after you understand virtual memory, then read this guide before you tackle the projects.
* You're reasonably familiar with (but not necessarily an expert on) C. I'm assuming you've used C at the level of something like CS 50, but not much more beyond that. If you haven't, don't try to learn C by reading the xv6 code or doing the OSTEP projects. Unfortunately, C isn't the kind of language you can pick up on the fly; it has way too many pitfalls. It's also a really bad idea to learn it from online tutorials; they're usually filled with mistakes. Use a book like [*C Programming: A Modern Approach*](https://www.amazon.com/C-Programming-Modern-Approach-2nd/dp/0393979504/), or *Modern C*. (Yes, I think [*The C Programming Language*](https://www.amazon.com/Programming-Language-2nd-Brian-Kernighan/dp/0131103628/) is great, but I also think it's too short and doesn't go over everything you need for xv6 and OSTEP.)
* You've taken a course like [Nand2Tetris](https://www.nand2tetris.org/) or [CS:APP](https://www.cs.cmu.edu/afs/cs.cmu.edu/academic/class/15213-f15/www/schedule.html) [(book)](https://csapp.cs.cmu.edu/3e/home.html), or you understand the basics of computer architecture and systems programming. You don't need to know any x86-specific details -- I'll talk about those along the way -- but it can't hurt either.

I recommend reading the posts in order; I'll assume you've read all the previous
posts in any later ones. Let me know if you find any errors or if you have any
suggestions to improve them!

Many thanks to [@spamegg](https://github.com/spamegg1) from [OSSU](https://github.com/ossu/computer-science/) for the many hours spent reading the source
code with me and helping me wrangle the OSTEP projects and improve their docs
and test scripts to work for other OSSU students!

## Contents

1. [Das Boot](boot.md)
2. [Entering the Kernel](entry.md)
3. [Detour: Spin-Locks](spin_locks.md)
4. [Paging: Page Allocation](page_allocation.md)
5. [Paging: Kernel Page Directory](paging_kernel.md)
6. Paging: User Space
7. Processes
8. It's a Trap!
9. Detour: Sleep-Locks
10. Devices: Multiprocessing
11. Devices: Keyboard, Serial Port, and Console
12. File System: Lower Layers
13. File System: Upper Layers
14. User Space: The First Process
15. User Space: The Shell
16. User Space: C Library and Utilities

## License

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
