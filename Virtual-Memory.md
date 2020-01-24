Virtual Memory
==============

Introduction
------------

At this point, your Weenix contains a threading library, some thin
wrappers around device drivers, and basic file system support with a
caching layer. By the end of this assignment, Weenix will be a full
operating system. With the addition of virtual memory, your kernel will
start managing user address spaces, running user-level code, and
servicing system calls. After completing this project, everything you
did before will seem insignificant.

This assignment is substantial, and also very prone to difficult bugs.
Before you begin, make sure the rest of your kernel is functioning
exactly as you expect. You will undoubtedly uncover bugs in old code
throughout the course of this assignment, but minimizing the number you
find before you start will be helpful. Make sure to start early and ask
questions frequently; it is very easy to get lost in this assignment.

Remember to turn the `VM` project on in `Config.mk` and `make clean`
your project before you try to run your changes.

Because VM bugs can spring up in code you wrote months ago, this is
where you will probably find out whether or not your implementations of
the previous assignments are up to par. We would like to point out that
there are several Weenix- and OS-specific debugging tools and techniques
in Appendix [Debugging] which will be *extremely* useful if you have not
been using them so far.

Virtual Memory Maps
-------------------

The first thing you should do in this assignment is write the code for
managing a process’ virtual address space. The virtual address space for
a process (also known as its “memory map”) is stored as a linked list of
virtual memory areas (also referred to as “memory regions”), each of
which correspond to some memory object which provides pages of memory to
the process on demand. As you have likely already realized, this means
that everything from files to disks can be mapped into the address space
of Weenix processes, and it should now make even more sense why we used
memory objects extensively in the last assignment instead of reading and
writing directly to disk. Of course, some memory areas will not
correspond to existing data (the stack and heap, for instance). We will
explain how that works in the section on .

In order to manage address spaces, you must maintain each process’ list
of virtual memory areas. Each memory region is essentially a range of
virtual page numbers, a memory object, and the page offset into that
memory object that the first page of the virtual memory area corresponds
to. Make sure that you understand why these numbers are all stored at
page resolution instead of byte (address) resolution. You must keep the
areas sorted by the start of their virtual page ranges and ensure that
no two ranges overlap with each other. There will be several edge cases
(which are better documented in the code) where you will have to unmap a
section of a virtual memory area, which could require splitting an area
or truncating the beginning or end of an area.

While there is very little conceptually difficult code to write in this
section of the assignment, off-by-one bugs are extremely common and
become very difficult to track down later on, so unit-testing this code
is a good idea.

Page Fault Handler
------------------

After your memory maps are working, you will need a way to actually load
the data into memory when a process attempts to access it. This is done
by the page fault handler. The page fault handler is triggered by a
processor interrupt when a process attempts to access an address for
which it has no lookup entry in the page table or the permissions on
that entry do not allow the type of access that is being attempted.

At this point in the project, any page faults that have occurred have
resulted in a kernel panic. That is because Weenix does not support
kernel-level page faults, meaning that the entire kernel address space
must reside in memory at all times. This functionality is written into a
wrapper for the page fault handler you will write which short-circuits
kernel page faults. More details on how this function works can most
easily be found in the code.

The combination of the page fault handler and the virtual memory maps
should be enough to get a very simple page fault to occur in a userland
program. At this point, you can set up a userland program to run from
inside the `init` process by running `kernel_execve()`, passing the path
to any program on your (virtual) disk as an argument. Similar to the
`exec()` system call, this will replace the memory map of the current
process to set up another program to run, but it will be better than
`exec()` in this case because the setup is done exclusively in kernel
space, so it can be used before you have a fully functional userspace.
When the program begins, it should cause a page fault to be generated.
This is your first step towards having a functional userland.

A simple implementation of the page fault handler will be enough to
start with, but eventually this will be a relatively logic-heavy
function. First, the page fault handler should search for the virtual
memory area containing the address that was faulted on. Then, the
permissions of this area should be checked against the flags variable
that is passed to the handler, which tells the handler whether the
attempted access is a read or a write. If the memory area containing the
accessed page is not found or the permissions would make this access
illegal, the current process is killed with an exit status of `EFAULT`.
Of course, if Weenix supported UNIX signals, it would send a `SIGSEGV`
signal instead.

Once the virtual memory area is found, Weenix must search for the
missing page and map it into the page table of the current process so
that the access can be retried using the virtual address of the page
that’s being added and the physical address where it resides. Fetching
the missing page will require a lot of help from the page frame caching
system, namely for looking up the page and dirtying it if the access is
a write. This, in turn, will rely on two new types of memory objects you
will need to implement.

### The Memory Management Unit

In order to map the virtual address to its corresponding page of memory,
you will need to use the page table functions. A good portion of memory
management is done for you, but you will have to fill in page table
entries when page faults occur, flush the translation lookaside buffer
(TLB) when necessary, and manage copy-on-write pages yourself. You will
also need to make sure that pages which are not backed by files remain
pinned, so they do not get paged out by accident.

Memory Management
-----------------

As you have implemented it currently, the caching layer of Weenix works
exclusively for pages of files or disks that have been mapped into
memory. You will extend it by creating memory objects which will provide
two new types of memory which are not backed by any on-disk structures.

### Anonymous Objects {#anon}

So far, you have used the memory objects of your block device and files
to fill page frames as you needed data from disk, but it does not make
sense to back some virtual memory areas, such as a process’ stack, with
data on disk. What you often want is objects which initialize pages by
filling them with zeroes and pin their pages in memory as long as the
process is using them. These are known as anonymous objects since they
are not backed by any persistent data (which would have a filename
associated with it). Anonymous objects are relatively simple to
implement, so look for a better description of how they work in the code
comments.

Notably, anonymous objects cannot be paged out in Weenix. The designers
chose not to implement this feature because memory pressure will rarely
be an issue in your operating system, and implementing a swap space is
not terribly interesting or vital to implement as a result.

### Shadow Objects

Anonymous objects are easy to implement, however you will also need a
much more sophisticated form of memory object called a shadow object to
implement `fork()`. These will be used to implement copy-on-write for
privately-mapped blocks that are accessed after forking. Because of how
involved shadow objects must be, you should refer to lecture slides or
the book for more general information about how and why they are used.
The rest of this section will only cover how to implement them in
Weenix.

To implement shadow objects, it will be extremely helpful to understand
how the methods of memory objects are called during a page lookup or
dirty operation. If you don’t remember this well from the last
assignment, we recommend that you go back and either re-read the
relevant sections of the last assignment or search through the code
paths in question and draw a graph showing what functions in the page
frame system call what functions in the related memory objects.

The main difference between shadow objects and other types of memory
management objects is that shadow objects can be part of arbitrarily
long chains of memory objects. Therefore, many calls to shadow objects
will be rerouted to the object that is being shadowed, or occasionally
to the root object in a tree of memory objects, which cannot be a shadow
object. At a high level, this is similar to how file memory objects
forward requests to the disk memory object, but in practice it ends up
seeming a lot more recursive when implementing shadow objects since
there is no translation layer as there was between file block numbers
and disk block numbers. However, shadow objects are still responsible
for storing some data and, more importantly, causing copy-on-write to
work after a `fork()` has taken place.

One potential problem with shadow objects is that the chains must be
cleaned up when the process that creates them exits to avoid temporary
memory leaks. Ideally, this could happen at process exit. A process
exiting might cause a shadow object’s refcount to drop to one, at which
point the pages attached to the object could be reassigned to the single
shadow object beneath it, and the object itself could be deallocated.
However, this would require the shadow object to know what its remaining
child is and, at the moment, shadow objects do not maintain a list of
their children.

This apparent design flaw leaves two other avenues for shadow chain
cleanup. First, there is a shadow daemon known as `shadowd` which was
built for this purpose. It should be invoked when the kernel is out of
memory (this code is already written) or when a shadow object which can
be cleaned up is created (you can tell this by checking for it when you
remove either of its child shadow objects). To enable the shadow daemon,
just set `SHADOWD=1` in the project environment settings. The shadowd
code exists as a testing tool, but you should *not* use it for your
final product.

The second method would be to collapse shadow chains during `fork()`.
This requires a relatively easy traversal of the forking process’ object
chains, where you shift the pages from any objects with a single child
down to their child and then deallocate the objects. You should
implement this for your final product inside your fork logic.

System Calls
------------

System calls are the only way user processes can communicate with the
Weenix kernel directly. The way that system calls are generated from
user space is by generating a software interrupt (using the x86 `int`
instruction) with the arguments to the system call stored in the
registers or on the stack. This causes an interrupt in the kernel, where
the number in an agreed-upon register designates which system call is
being used, and then the corresponding system call function is actually
called to handle the request after the arguments have been parsed out of
their registers. Most of the system calls have already been written for
you, however, in order to give you some understanding of the process
involved, you will need to write a few yourself.

### Kernel System Call Interface

You will need to implement the kernel targets for `read()`, `write()`,
and `getdents()`. While most of what you need to do should be pretty
self-explanatory after reading through other system call
implementations, you must also write two helper functions to check
accesses to user memory from within the kernel.

### Accessing User Memory

The code to handle traps and access user memory from the kernel has been
written for you. However, many of these functions need to check to see
if a region of user memory is a valid section of the process’ address
space. To check this, you will need to implement `addr_perm()` and
`range_perm()`, which will rely on your virtual memory map code.

### Running Userland Programs

Once you have implemented the page fault handler, anonymous objects, and
`write()`, you should be able to run a variety of simple user-level
programs. Of course, the first you should try to get running should also
be one of the simplest, so we recommend `hello`, which should print
“Hello, world!” to the screen. To run correctly, this will require a
mostly-functional page fault handler to fill in pages as the process
attempts to access them; otherwise, the operating system will probably
go into an infinite loop, trying to access the same address over and
over using the page fault handler, but never adding the correct entry to
the page table. Some other simple programs that you should be able to
run are `args` and `uname`.

If you are having trouble getting `hello` to run and suspect that your
anonymous object or `write()` implementations might be at fault, you
should try the `segfault` program instead and ensure that it exits with
a status of `EFAULT`. If it doesn’t (if, for instance, you run into the
infinite loop problem described above), this means the bug is probably
in your page fault handler.

After getting `hello` or `segfault` running, congratulations! You’ve
just gotten your first userland program working! Celebration techniques
are myriad, but we recommend dancing around a bit, and maybe taking a
shower.

At this point, it will be useful to look at the appendix covering how to
inspect the progress of a user-level process using a debugger. Although
you may not need it yet, we assume that you will want it very soon.

### VM-Related System Calls

After you get some initial test programs running, you can start to think
about implementing a variety of VM-related system calls. For the
functions in this section, we recommend that you check out the
documentation in the `man` pages for more information.

`mmap()` and `munmap()` are the most simple and obvious of the functions
in this category. They allow user processes to map files into memory,
create private or shared memory regions, and remove areas of their
address space. The majority of these functions will end up being
error-checking, since you wrote the main logic for them in the virtual
memory map code. Note that the Weenix memtests expect you to use the
`VMMAP_DIR_HILO` flag.

`brk()` is similar in conceptual difficulty. Calling `brk()` changes the
length of the memory region acting as the heap, but the pointer passed
as an argument to `brk()` is not required to lie on a page boundary, and
the beginning of the heap sometimes starts halfway through the last page
of another memory region. This means that the edge cases for `brk()` can
be a bit annoying, but there’s nothing conceptually difficult to grasp
here. There are some robust user-level tests for much of this
functionality, so rather than spending a lot of time getting it right
before testing, we recommend starting with something naive and gradually
fixing it to pass the tests after you can run them in userland.

`fork()`
--------

Although it is also a VM-related system call, `fork()` is an entirely
different animal from `mmap()` and friends. A good implementation of the
previous sections is essential; `fork()` is complicated enough without
having to debug the rest of your VM code at the same time. The `man`
pages, while useful as always, will not be as helpful for `fork()` as
for the other system calls, so most of the documentation for `fork()` is
given here.

`fork()` is a moderately complicated system call. We present it here as
one long algorithm, but it will make your life much easier if you break
it down into separate subroutines. Close attention to detail will help
you; an under-debugged `fork()` can cause subtle instabilities and bugs
later on.

Bugs in the virtual memory portion of `fork()` tend to cause bizarre
behavior: user process memory may not be what it ought to be, so almost
anything can happen. The user process may end up executing what should
be data, jumping into the middle of a random subroutine, etc. These
sorts of bugs are *very* difficult to track down. For this reason, you
should code more defensively than you may be used to. Assert everything
you can, `panic()` at the first sign of trouble, and include apparently
unnecessary sanity checks.

Above all, be sure you really understand the algorithm before you start
coding. If you try to implement it before you understand what you are
trying to do, you will write buggy code. In all likelihood you will then
forget that you have written buggy code, and waste time debugging code
that you should have thrown away. We know this because it has happened
to us.

Note that these steps are not all in the correct order; consider the
order in which you do them, keeping in mind what kind of cleanup you
will need to do if one of them fails. Look out for steps which cannot be
undone.

-   Create a new process using `proc_create()`.

-   Copy the `vmmap_t` from the parent process into the child using
    `vmmap_clone()` (which you should write if you haven’t already).
    Remember to increase the reference counts on the underlying memory
    objects.

-   For each private mapping in the original process, point the virtual
    memory areas of the new and old processes to two new shadow objects,
    which in turn should point to the original underlying memory object.
    This is how you know that pages corresponding to this mapping are
    copy-on-write. Be careful with reference counts. Also note that for
    shared mappings, there is no need to make a shadow object.

-   Unmap the userland page table entries and flush the TLB using
    `pt_unmap_range()` and `tlb_flush_all()`. This is necessary because
    the parent process might still have some entries marked as
    “writable”, but since we are implementing copy-on-write we would
    like access to these pages to cause a trap to our page fault handler
    so it can dirty the page, which will invoke the copy-on-write
    actions.

-   Set up the new process thread context. You will need to set the
    following:

    -   `c_pdptr` - the page table pointer

    -   `c_eip` - function pointer for `userland_entry()`

    -   `c_esp` - the value returned by `fork_setup_stack()`

    -   `c_kstack` - the top of the new thread’s kernel stack

    -   `c_kstacksz` - size of the new thread’s kernel stack

-   Copy the file table of the parent into the child. Remember to use
    `fref()` here.

-   Set the child’s working directory to point to the parent’s working
    directory. Once again, don’t forget reference counts.

-   Use `kthread_clone()` (which you should write if you haven’t yet) to
    copy the thread from the parent process into the child process.

-   Set any other fields in the new process which need to be set.

-   Make the new thread runnable, which will add it to the run queue.

Remember that the only difference between the parent and child processes
is the return value of `fork()`. By 32-bit x86 convention, this value is
returned in the `eax` register, which should be set in the context
values of both threads. You should also revisit your implementation of
the `proc_exit()` function to make sure that your implementation is
releasing all resources it should.

Note that a simpler, less correct implementation of `fork()` can
function without actually using shadow objects, as long as you don’t
care what happens to whichever process (parent or child) wakes up last
from the syscall. If you’re having trouble getting shadow objects to
work correctly, you can write `fork()` without them for testing
purposes.

Odds and Ends
-------------

Finally, there are a number of other functions which you might remember
seeing in earlier assignments spread throughout the kernel which you
need to find and either write or update. These functions are all fairly
small, but if you miss one, some things will break. Two examples are
`special_file_mmap()` and `proc_kill_all()`. Once you get the last of
these finished, you should be able to test your kernel with any binary
file you find on the Weenix file system.

Testing
-------

Testing your code at this point becomes rather difficult, since you must
be able to create data and text in user land and execute it. This is an
order of magnitude more difficult than creating kernel-mode threads as
you have in past assignments. Thankfully most of the gory details have
been taken care of for you (take a look at `kernel/api/elf32.c` and
`user/ld-weenix/` if you are a masochist).

### Userland Tests

Once you have functioning userland execution and a working `fork()`
function, you are ready to complete your Weenix system by running the
userland binaries we provide for you. All you need to do is call
`kernel_execve()` in your init process. You should execute the binary
`/sbin/init`, which should start 3 shells (one in each terminal window).
These shells will allow you to execute any of the provided binaries
(roughly in order of difficulty):

-   `/usr/bin/segfault` - Even simpler than hello, this should just
    segfault on address `0x0`. Good if you’re having a lot of trouble
    getting hello to run.

-   `/usr/bin/hello` - A simple “Hello world!” test. Getting this to
    execute properly should be a big step in VM.

-   `/usr/bin/args` - Prints command arguments.

-   `/usr/bin/forktest` - Simple program which forks, and prints out
    everything of note.

-   `/bin/uname` - Prints system information.

-   `/bin/stat` - Prints information about a file.

-   `/usr/bin/kshell` - Traps into the kernel and starts a kshell.

-   `/bin/ls` - List the contents of a directory.

-   `/sbin/halt` - Kills all processes and shuts the system down.

-   `/usr/bin/wc` - Counts characters, words and lines.

-   `/bin/hd` - Dumps input in hexadecimal.

-   `/bin/sh` - The shell itself. Yay subshell fun!

-   `/usr/bin/spin` - Executes “`while(1);`”.

-   `/usr/bin/forkbomb` - A forkbomb test which should theoretically run
    forever.

-   `/usr/bin/stress` - A test to stress various parts of your system
    and then run a forkbomb.

-   `check` - Contains checks for various test cases (this is a shell
    built-in command).

-   `/usr/bin/vfstest` - Lots of VFS tests (error conditions, etc.).

-   `/usr/bin/memtest` - Lots of memory management tests (mmap and brk).

-   `/usr/bin/eatinodes` - Devours filesystem resources.

-   `/usr/bin/eatmem` - Devours kernel memory.

-   `/bin/ed` - `ed` is the standard text editor.

The shell also has a bunch of builtins. Type `help` in a shell to see a
list of them. In particular, `repeat` and `parallel` can be very useful
for stress testing your kernel.

### A Relatively Difficult Test Suite

In addition to just having commands which work individually, you should
be able to stress the hell out of your system. Run lots of difficult
commands (`forkbomb`, `eatmem`, etc.) simultaneously, use different
terminals at once, `halt` in the middle of all this, and so on. This
type of testing can frequently be quite random, so here is a more
systematic list of some things you can try. Make sure you start with a
fresh disk.

-   `cat hamlet`

-   `cat hamlet > hamlet2`

-   `halt` (to shut down, then make sure the changes still exist on disk
    when you reboot)

-   `rm hamlet`

-   `cat /dev/null > foo`

-   `ln foo bar`

-   `cat README > foo`

-   `cat bar`

-   `check all` (do this three or four times in a row)

-   `vfstest`

-   `memtest` (do this three or four times in a row)

-   `parallel vfstest –- vfstest`

-   `parallel memtest –- memtest`

-   `vfstest` and then `halt` while running (use `repeat` to re-run
    `vfstest` if it finishes too quickly)

-   `memtest` and then `halt` while running

-   `forkbomb` and then `halt` while running

-   `forkbomb` and then `eatmem`

An easy way to make these tests harder is to check the kernel memory
allocators for any leaks. See the appendix to see how to do this. You
may also want to test what happens when Weenix runs out of disk data
blocks. To do this, set the `DISK_BLOCKS` variable to 2048 if it is not
already, and then re-run Weenix with a fresh disk and execute the
commands below.

-   `cat hamlet >> hamlet` (this should reach the maximum file size and
    then exit)

-   `vfstest` (this should work - the disk is not yet filled)

-   `cat README >> README` (this should fill the disk but not quite
    reach the maximum file size)

-   `vfstest` (this should fail to run most of the tests due to no disk
    data blocks being available)

### Dynamic Linking

Once you feel everything is in good shape, enable the `DYNAMIC` variable
and recompile the project from scratch. This will cause your userland
libraries to be dynamically linked, which puts much more stress on your
VM (especially `mmap()` code). This essentially adds another layer of
indirection between the executable being run and the library calls it’s
attempting to run, where `ld-weenix` can link the library calls that are
used into the original binary at runtime. Unfortunately, this also makes
it even more difficult to debug what the user process is doing; if you
are interested in setting breakpoints in the user process with dynamic
linking, check out the appendix for more information.

Turning dynamic linking on will make the above tests even more thorough.
For instance, the test using `forkbomb` and `eatmem` simultaneously is
notoriously hard to get right in the presence of dynamic linking - a
phantom bug might cause your `pageoutd` to thrash back and forth between
two pages if you’re not careful.

Conclusion
----------

We hope you’ve enjoyed working on Weenix and that you learned a lot.
Good luck finishing your project, and don’t forget to read the appendix
if you run into problems or need more ideas about where to look!
