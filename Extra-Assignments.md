Extra Assignments 
=================

The features listed on this page are not extra credit. **There is no
extra credit in Weenix.** Your grade is based on how well you complete
the core Weenix requirements. Therefore trying to implement these things
can only hurt your grade and distract you from what is really important
in life. The only thing to be gained by foolishly ignoring this warning
is a deeper understanding of Weenix and bragging rights. **We will look
unfavorably on someone who has implemented some of these, but has
problems with the core Weenix projects.**

Some items on this page include descriptions on when it is feasible to
implement them, be aware however that you should always keep around a
copy of your work which does not contain your work on extra features in
case you break something. **Having a broken Weenix because you tried to
implement one of these features is not acceptable.** This is
particularly important to remember because you might implement a feature
between and only to find later when working on that you broke something
badly.

Also note that the course staff will only be able to provide limited
help with these features. Some of the features have been implemented in
the staff version of the code, others have been attempted and abandoned,
still others are random whims which may or may not be impossible.

Realistic projects
------------------

If you ignored the warning above then this is the place to start. These
are features which are known to be possible because someone has either
done so or it has been planned. This means you will get some description
here on how to implement it and there might even be helpful code already
in Weenix.

### Multithreaded processes

This task has a bunch of places in code that are marked by the `__MTP__`
symbol (which must be enabled by setting “`MTP = 1`” in `Config.mk`),
and the kernel is designed around it anyway. This is a relatively
straightforward addition to the kernel.

### Current working directory

This will add a system call which looks up the full path name of the
current working directory of the current process and is marked in the
codebase with the symbol `__GETCWD__`, which can be enabled by setting
“`GETCWD = 1`” in `Config.mk`. This should not be too difficult at all.

### File system mounting

One important missing feature in our VFS implementation is mount points.
We have a root file system (`ramfs` if you are working on , `s5fs` if
you are working on ), but normal UNIX allows you to access multiple file
systems by “mounting” additional file systems on directories in your
virtual file system layer.

For more information on mount points we refer you to the `mount(2)` man
pages, the `umount(2)` man pages, the lecture slides, and the text book.

Before you begin, there is already a significant amount of code in
Weenix for this feature; however, by default it is not compiled. To
compile this code change the line “`MOUNTING = 0`” in `Config.mk` to
“`MOUNTING = 1`”. Also, remember to run:

    $ make clean

You should use `cscope` to search for all instances of the C symbol
`__MOUNTING__` in Weenix to see exactly what has changed to allow
mounting to happen. Notice that `struct fs` in `fs/vfs.h` now has a new
field called `vn_mount`.

The biggest change caused by changing the `MOUNTING` flag is the
behavior of `vget()` (even though very little code changed, the behavior
is completely different). The easiest way to think of the behavior of
the new `vget()` is like this:

    vnode_t* new_vget(args) {
        vnote_t* vn = old_vget(args);
        if (!error) {
            return vn->vn_mount;
        } else {
            return error;
        }
    }

The purpose of this change is to make the integration of mounting as
seamless as possible. Normally `vn->vn_mount == vn` and therefore most
of the time this new behavior is identical to the old one; however,
simply by setting the `vn_mount` field the `vget()` function
automatically traverses into mounted file systems for us. The only cases
we need to worry about are when we are leaving a mounted file system
(e.g. following a `..` path from the root of a mounted file system).

The easier part of implementing mounting is filling in some functions
which have been left blank (but fully commented!) for you. In `fs/vfs.c`
this is:

    int vfs_mount(struct vnode *mtpt, fs_t *fs);
    int vfs_umount(fs_t *fs);

in `fs/vfs_syscall.c` there is also:

    int do_mount(const char *source, const char *target,
                 const char *type);
    int do_umount(const char *target);

You will also want to read the Hackers Guide section about reference
counting for information about special conventions used when reference
counting mount point `vnode_t`s.

Now comes the hard part: implementing these functions is not enough. As
was noted in the previous section, setting up the `vn_mount` field will
allow us to enter mounted file systems. However following `..` paths out
of mounted file systems still needs to be special cased. You will need
to think about the code you wrote for and which code will need to be
different in order to handle mounting correctly. The amount of code you
will need to write is small, but you need to find the right functions to
write it. All of the code should go into functions which you wrote for .
You do not need to modify functions which you wrote for other projects
or functions which you did not write originally. Also remember some of
the system calls have errors specific to mounting which you might not
have worried about before (e.g. What happens when you try to link a file
from one file system onto another? Why is this an error?).

### Userspace preemption

Definitely possible. The basic idea is that a timer interrupt is
scheduled to occur every several milliseconds. The interrupt context
then looks at the thread which was interrupted. If the thread was in
userspace it is safe to just put that thread on the run queue and switch
into the context of another thread (thus preempting the userland
process). If the interrupted thread was in the kernel, we do not want to
arbitrarily preempt it (preemptible kernels lead to kernel hacker hell).
Instead, we set a flag on the thread to mark it as preempted and allow
it to continue. When a thread returns from kernel land into user land
the flag should get checked and if it is set the preemption should
happen at that point. Make sure you understand all of this before you
start or things will get messy.

So, do you gain anything from doing this? Actually, the effects are very
visible and very satisfying. You will be able to run `vfstest` and
`cat hamlet` at the same time and they will all look like they are
running at the same time (assuming you set a good preemption time) while
your fellow students will have very visible times when one operation
stops working while the other is running. Also, your Weenix will not
hang while you `cat /dev/zero` to `/dev/null`.

To get started on this enable the compile time flag and check out the
code in `kernel/util/time.c`. It sets up the timer interrupts but you
have to fill in what to do once they happen.

### Pipes and synchronous multiplexing

One thing that greatly increases the power of shells and other programs
is pipes, because they all for simple inter-process communication. This
is not difficult to add to the standard Weenix, and some support code
has been included to help you. Enable the pipe subsystem by setting
`PIPES = 1` in `Config.mk` and look in `fs/pipe.c` to get started. Make
sure you understand how the `pipe(2)` system call works. It has the
following signature:

    int do_pipe(int pipefd[2]);

If successful, this call will fill the supplied `pipefd` array with a
file descriptor representing the end of the pipe for reading, and then a
file descriptor representing the end of the pipe used for writing.

Once you are done with that, you have a use case for synchronous I/O
multiplexing, which is where a program can wait for any of multiple file
descriptors to become ready. This is typically implemented using the
`select(2)` and `poll(2)` system calls. Look those up for more
information, but the basic premise is that you want to check if any of
the file descriptors the user put in are ready for whatever operation
they requested, and if they are not, have some way to sleep until one or
more of them is ready.

These syscalls are filesystem-oriented and rely on some extensions to
`vnode` operations that you won’t have to touch for normal Weenix. For
example, because pipes have to know when all of the readers of a pipe
have closed their file descriptors, the pipe must keep an accurate count
of how many files have it open for reading and writing. This requires
new `vnode` operations that also take the `file` which is getting a
reference (called `acquire`) or putting a reference (called `release`)
to the `vnode`. Requesting events from `vnode`s for multiplexing will
require an operation typically called `poll`, which checks whether
reading or writing will block, among other things.

### Asynchronous disk driver

It would be cool to support an asynchronous disk driver. To do this
would require a decent amount of restructuring in the driver code, but
is doable given a bit of time and effort. After this is done, you could
potentially add asynchronous user I/O support as well (although this is
a harder problem because in most Unix distributions the notification of
a completion is done through signals).

### Better scheduling

After you have finished implementing processes and threads, you should
know the basics, but it might be a bad idea to try making too many big
changes to Weenix before you have really gotten exposed to everything.
The current Weenix scheduler is rather primitive as it has no sense of
priorities and makes no attempt to do any “clever” scheduling; it is
purely first come first serve. Here are some helpful suggestions:

-   Professor Doeppner’s lecture (only available to Brown students) on
    scheduling contains plenty of information on different scheduling
    algorithms (you might also check the textbook).

-   The CS167 students implement a slightly more advanced scheduler for
    their threading library assignment. You might be able to get some
    inspiration from that handout or the support code.

-   If you decide to pursue user space preemption you might be able to
    tie it in nicely with a more advanced scheduler (e.g. threads which
    have been running for a long time and keep getting preempted should
    get a lower priority than threads which have just recently woken up
    after being asleep. As described in the lectures on scheduling, this
    can help because the long running programs tend to be background
    processes, while things which have just woken up are probably user
    processes which just received input).

Most of these have not been implemented yet, but a simplistic O(1)
scheduler has been implemented in the past with interesting results. The
downside is that it will probably not have very visible effects, but you
might be able to think of a good way to show off its effects that we
have not.

Possible long-term projects
---------------------------

If you are done with Weenix and you really loved it, you might want to
consider a larger addition to Weenix to enrich your own knowledge of how
operating systems work. These projects are almost certainly not possible
to implement in the same semester you are working on Weenix if you are
taking it as part of a class, but they represent interesting areas of
potential further study.

### Multi-core/processor support

There are a few issues which make the jump to multi-processor land
somewhat painful. One is that the boot process must be overhauled almost
completely to compensate for the extra processors. You must learn about
the boot process in the x86 architecture – this will require intimate
knowledge of ACPI, inter-processor interrupts, the interaction between
CPUs, I/O APICs and Local APICs, and some knowledge of real-mode 16-bit
x86 assembly. Setting up page tables correctly becomes more difficult as
well. You will also have to learn about the global descriptor table
(GDT) and the task state segment (TSS). (Be warned that once you are
done with this step, you will probably hate Intel for the rest of your
life, and also know the x86 architectyure better than anyone you know.)

Once you get the other processors to boot, get into protected mode with
paging enabled, and then make the jump into the kernel binary, you will
have to get your processors to do something more interesting than spin.
This will require a better scheduler, because the usual mechanism for
getting work off your boot-strap processor and onto your other
processors is to have the idle processors steal it. Work-stealing can be
done simply, but the best schemes are rather complicated.

In addition, you will now have to worry about the SMP-safety of your
kernel. Weenix is certainly not SMP-safe by design, and a lot of
portions of the kernel make assumptions that are fine when they are the
only thread running but which quickly deteriorate in the multiprocessor
world. A library of synchronization primitives will be needed. The
simplest way to make the kernel SMP-safe is to use a Big Kernel Lock,
which was done back in old versions of Solaris and Linux. While this is
a decent starting point, this will lose most of the benefit of
implementing a multi-processor system, as you will only get any real
parallelism as long as the programs are running in userland.

If you really want to implement this, talk to Eric Caruso or Jackson
Owens. This is probably best done along with multi-threaded processes
and userland preemption, as well.

### Users

The interesting part about this extension is not necessarily the
implementation of users themselves, but the security model that you will
have to implement for it to be meaningful. You’ll have to write a new
file system which has some form of access control. In standard Unix,
this is based on users and groups, but in Windows, something more like
ACLs are used instead. You may want to experiment with
capabilities-based forms of security if you want a more novel project.

### Signals

Signals are cool, but they get pretty ugly pretty fast when you start
doing stack manipulations in assembly. As long as you have a fairly
strong idea of x86 stack discipline, it will be more messy than
difficult.

“Abandon all hope, ye who enter here”
-------------------------------------

If you ignored the Dante quote above then this is the place to start.
These are the features that we know are either very difficult or
completely impossible. However, we would hate to stop a truly foolish
individual from banging their head against the keyboard until one of
these features pops out.

### Networking stack

If you’ve taken a networking class where you implemented TCP/IP, you
could try this one out. However, before you start, don’t forget to write
an Ethernet driver from scratch. Once you’re done, demonstrate by
porting Apache or nginx to Weenix.

### X window system

Depends on signals, graphical display driver with higher resolution than
$80 \times 25$, mouse driver, etc. You’ll also need a pseudo-terminal
subsystem, which will be a huge pain.

### 64-bit support

This isn’t even useful because the Weenix kernel has no way of allowing
a process to address more than 1GB of memory as it is. (Think about how
paging works and you will figure out why.)If you want it to be able to
support large amounts of memory, be prepared to rewrite all of the
paging code, all of the assembly, and probably a lot of other random
stuff.

### Kernel preemption

Would require totally redesigning the Weenix kernel to make it
threadsafe. If this seems reasonable, you are misunderstanding
something. Userspace preemption is similar and doesn’t require a full
rewrite of Weenix. Also note that most production kernels don’t even
support this (because even after you get it working, it’s an absolute
nightmare to maintain, and it typically adds very little value).
