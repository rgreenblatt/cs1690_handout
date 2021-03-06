Debugging 
=========

As you begin to develop your kernel, you will undoubtedly find problems
in the code you have written. We have collected some techniques here
which we found useful or which are largely undocumented elsewhere
because they are fairly advanced or are specific to working on Weenix in
particular.

Beginning Debugging
-------------------

These are the simplest debugging techniques, which can be used at any
stage of the project. They require the least amount of learning, but
they also are not nearly as powerful as the debugging techniques listed
later on.

### Printing

By far the easiest technique for debugging is printing things out during
the execution of your program. Of course, printing inside the kernel
requires significant overhead (as you will learn when you write the TTY
driver). Luckily, inside the support code we have provided printing
methods which use the serial port of the virtual machine to print to a
terminal outside of the computer. You can think of this as being like a
printer connected to the serial port if you wish. The way to use this is
through the `dbg` or `dbgq` macro.

    #define dbg(dbgmodes, printfargs)
    #define dbgq(dbgmodes, printfargs)

The difference between the two is that `dbg` additionally displays the
file and line number information describing where the debug statement is
located, but `dbgq` displays only the debugging message.

Debug messages are organized into many debug modes, each of which can be
separately hidden or color-coded in the debug output. You can find a
list of debug modes in `kernel/include/util/debug.h`. The string names
next to each mode are used to configure which debug modes are displayed
(the default can be set in `kernel/util/debug.c` and updated
while Weenix is being debugged via the dbg command), and there is also a
special string name which refers to all debug modes. For example:

    dbg(DBG_THR, "Creating a kernel thread "
                 "with stack at 0x%p\n", stack);

In the snippet above, `DBG_THR` is the mode macro for the threading
subsystem. This debug message would print something like:

    kthread.c:123 kthread_stack_create(): Creating a kernel thread
    with stack at 0x804800

It is also possible to define a debugging message which is part of
multiple modes by taking their bitwise or.

    dbg(DBG_THR | DBG_VM, "Creating a kernel thread with stack at "
                          "0x%p\n", stack);

It???s not hard to add your own debugging modes, either. This is helpful
if you decide to implement extra features and would like an easy way to
separate their debug output from the normal subsystems without turning
all of the debug modes off.

### Printing Using Info Functions

There are several info functions in the kernel which are provided
exclusively for visualizing information in the debug console. To call
one of these, use `dbginfo()`, like so:

    dbginfo(DBG_VMMAP, vmmap_mapping_info, curproc->p_vmmap);

You can substitute in other functions besides `vmmap_mapping_info` -
there are a bunch of functions ending with ???`_info`??? in Weenix which can
be passed here.

### Using Assertions

Another simple but widely-applicable approach is to intentionally crash
your program when some condition is not met, which is known as
???asserting??? that condition???s truth. This allows you to know what the
failure point was, which is sometimes helpful for figuring out what
caused the failure in the first place, but mostly it???s helpful just for
letting you know that there was a failure instead of failing silently.

We provide assert functionality with the `KASSERT()` macro, which tests
the condition you pass it and prints out an error message on crash that
tells you the condition that failed and where in the codebase the
`KASSERT()` was. A trick you may find helpful is that any string will be
evaluated to true in C (because it???s just a pointer) so you can do
things like this:

    KASSERT(a == 1 && "A should be equal to 1");

This may be more helpful than not using a string, since the message will
be printed when the assertion fails. Of course, you can also just put a
comment next to the assertion in the code.

Because assertions are helpful for checking things that should always be
true, we recommend placing them at the beginning of any function whose
arguments must be of a certain form to ensure that the caller is
checking and filtering all error cases correctly (this is especially
helpful for syscalls). Don???t confuse this with error-checking that you
need to do, though ??? if you use assertions to check for error conditions
you are supposed to handle, you will needlessly cause your kernel to
panic!

Intermediate Debugging
----------------------

Although the techniques above are very useful, it will quickly become
apparent that a more robust way to debug the kernel is frequently
necessary. Luckily, debuggers provide us with an excellent way to do
this. `gdb` is the definitive debugger for C code on Unix systems, and
the rest of this appendix will center on its usage. Most of the
following information is applicable to all stages of Weenix development,
except where noted.

### Prerequisites

If you have never used `gdb` before, we recommend finding online and
reading it. Before you read more advanced or Weenix-specific debugging
tips, make sure you can use the following `gdb` features (most important
ones listed first, more specialized ones later).

-   Compiling with symbols.

-   Breakpoints, stepping, continuing.

-   Viewing and traversing the call stack.

-   Inspecting variables.

-   Inspecting the contents at a particular memory address.

-   Using watchpoints.[^1]

-   Conditional breakpoints.[^2]

-   Inspecting the contents of registers.

Once you understand these basics, you should be able to debug pretty
much any simple user-level program that you have the code for. However,
there are a few more specific topics that are helpful in some cases.

### Debugging Multiple Threads

Although this does not relate directly to Weenix, you will frequently be
debugging multithreaded programs. Debugging in the presence of multiple
threads is usually the same as debugging single-threaded programs, but
the place where it differs is in cases like deadlock. If you are
debugging a program in `gdb` and it deadlocks, you can hit `Ctrl-C` to
pause the program. Then, you can inspect the stack trace as usual. If
you need to check the other threads (perhaps the first one you were
given wasn???t the one in a deadlock, or maybe you want to see what other
threads are contributing to the deadlock) you can see a list of threads
using ???`info threads`??? and can switch to thread `N` by typing
???`thread N`??? (substitute the real thread number for `N` in that
command).

Another note about multiple threads (particularly if you might be
canceling them) is that you must be careful what calls you make during
critical sections of code. For instance, if you want to ensure that a
call to cancel some thread doesn???t take effect until after a certain
section of code is complete, there is a set of standard library calls
which make system calls which might act as a cancellation point. Look in
the `man` page for `pthread_cancel()` for a list of these library calls.
Note that `printf()` is one of them, so you might want to use `gdb` to
debug instead of printing out messages to tell you when certain events
happen.

### Using the Weenix `gdb` Scripts

Because Weenix has been debugged using `gdb` for some time, there are a
few Python scripts which run as custom commands inside the debugger
which you can use to help debug your OS. These are all given under the
command ???`kernel`???, and you can find more detailed information about the
commands available by running ???`help kernel`??? inside of `gdb`, but here
are a few of the highlights.

-   To turn kernel memory checking on, add the line
    ???`set kernel memcheck on`??? to the beginning of `init.gdb`. This
    allows you to run `kernel page` and `kernel slab` at shutdown, which
    tell you how much memory has not been cleaned up. Note that turning
    memory checking on may slow Weenix down a little bit, and that a
    handful of memory segments simply cannot be freed (page tables and
    stacks are common culprits).

-   You can access the debug info functions mentioned above through the
    `kernel info` command.

-   `kernel list` will tell you all the elements of a linked list if you
    used the macro list implementation.

-   `kernel proc` will give you a list of all live processes.

Note that these require a certain version of the Python language and
`gdb` to run; an error will be printed at the beginning of your `gdb`
sessions if your versions are not compatible.

### Disassembling a Program

If you need to know what exact instructions are being run or where
exactly in your code you are executing, you should probably disassemble
your program. The easiest way to do this inside `gdb` is to use the
`disas` command, although you can also use a command like `x/100i $eip`
to print the next 100 instructions starting at the address pointed to by
your instruction pointer. You can also use the `objdump` tool (separate
from `gdb`) to disassemble the entire contents of an executable.

### Using the QEMU monitor

Running the command `weenix -d qemu` will launch the QEMU monitor. From
here, you have direct access to more system level information than you
would in gdb. Notably, the `xp` command will let you examine physical
addresses, much like how x will still allow you to examine virtual
addresses. This can be useful in VM debugging to ensure the contents of
a physical address are what they should be. Running the `help` command
produces a full list of commands.

Advanced Debugging Techniques
-----------------------------

Finally, there are some times when the approaches above are just not
enough. These tips are for Weenix in particular, although they can
easily be adapted for use in debugging many other kernels as well. These
techniques will only be relevant when working on the virtual memory
system of Weenix, since they are mainly for debugging problems in
userland processes.

### Debugging a Page Fault

The most obvious thing to do when debugging a page fault is to look at
the address the fault is happening on. However, this doesn???t tell you
the context under which the page fault was generated, such as the stack
or the section of code that was running. To find these, the easiest way
is to inspect the stored context of the process that generated the
pagefault. This will allow you to see the address of the instruction
which was running (and perhaps more importantly, the name of its
containing function), the stack pointer, and any other registers which
might be needed to figure out what the processor was doing. This trick
comes in especially handy when you also load in the symbol file for the
user-level process, as the next section will show.

### Debugging processes from the kernel debugger

It is a little bit like black magic, but you can actually debug user
processes from inside the kernel debugger if you load in the symbol file
for the user process into the correct location in memory. To find this
information, you can use `objdump` (in your ordinary shell) to get some
information about the user process.

    $ objdump --headers --section=".text" user/simple/hello

    user/simple/hello:  file format elf32-i386

    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      6 .text         00000064  08048208  08048208  00000208  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, CODE

The relevant information that this gives us is the starting address of
the text section, `0x08048208` (the `VMA` column). So, we can open up
Weenix with `gdb` attached, and add in the relevant symbol file for our
userland process.

    Breakpoint 3, bootstrap (arg1=0, arg2=0x0) at main/kmain.c:121
    121     {
    (gdb) add-symbol-file user/simple/hello 0x08048208
    add symbol table from file "user/simple/hello" at
            .text_addr = 0x8048208
    (y or n) y
    Reading symbols from user/simple/hello...done.
    (gdb) b main
    Breakpoint 4 at 0x8048208: file ./hello.c, line 12.
    (gdb) c
    Continuing.
    ...
    Breakpoint 4, main (argc=1, argv=0x8047eec) at ./hello.c:12
    12      {
    (gdb) list
    7
    8       #include <unistd.h>
    9       #include <fcntl.h>
    10
    11      int main(int argc, char **argv)
    12      {
    13              open("/dev/tty0", O_RDONLY, 0);
    14              open("/dev/tty0", O_WRONLY, 0);
    15
    16              write(1, "Hello, world!\n", 14);

Now, you should be able to set breakpoints in the userland process.

It is worth noting, however, that this does not prevent you from also
putting breakpoints in kernel code. `gdb` is intentionally dumb about
how breakpoints work - whenever your instruction pointer reaches the
specified address, `gdb` will pause, no matter what symbol files you???ve
added - so since the text of your kernel and the user process are both
loaded, you can place breakpoints in either one.

### `gdb` and `DYNAMIC`

With dynamic linking enabled, it becomes a step more difficult to debug
userland processes from the kernel debugger, but certainly not
impossible. The main issue is that there is a bunch more code in the
`ld-weenix` shared library that you won???t be able to debug unless you
tell `gdb` to load the symbol file. The best way we know of to do this
is to print out the memory map of the process you???re trying to debug and
make an educated guess about which region might correspond to
`ld-weenix`. (It should be a shared region with execute permissions.)
Then, load the debugging symbols starting at the address which
corresponds to the beginning of that region.

[^1]: Watchpoints do not work with all the simulators which Weenix will
    run on.

[^2]: Conditional breakpoints do not work with all simulators which
    Weenix will run on. To get similar functionality, add an `if`
    statement to your code with the condition you care about and set a
    breakpoint inside there.
