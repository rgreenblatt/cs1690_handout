# Processes and Threads
## 3.1 Introduction
This assignment, colloquially referred to as “Procs”, will provide the basic
building blocks for the Weenix operating system: threads, processes, and synchronization primitives. These objects are implemented fully in kernel code, and interactions with user-level processes and threads will not be possible until you implement virtual memory in a later assignment.

Writing an operating system can be complicated, so much of the code to do
basic kernel operations such as memory allocation and page table management
has already been written. These are either beyond the scope of the project or
are too time consuming for too little value. While modifications to the support
code are fully possible, significant changes to the provided interfaces make errors
far more likely and make it harder to ask for help. However, it is important for
you to be able to take ownership of the code. Though it is usually too timeconsuming to write the entire system from scratch, by the end of the project,
you should have a good (if high-level) understanding of all the code involved.
This will certainly be true by VM, which will involve putting the final touches
on the kernel.

At the end of this first assignment, the kernel will be able to run several
threads and processes concurrently in kernel mode. It is important to emphasize
that a strong test suite is critical. For this assignment, test code must be added
directly into the boot sequence for the kernel, but in later assignments there
will be several additional ways to add tests in a cleaner, more modular way.

## 3.2 Kernel Memory Management
As mentioned above, this has been fully implemented for you, but take a moment
to look around `kernel/include/mm/kmalloc.h`, `kernel/include/mm/slab.h`
and `kernel/mm/slab.c` to familiarize yourself with the memory management
interface. The functions `kmalloc()` and `kfree()` exist as kernel-space work-alikes
for the standard library `malloc()` and `free()` functions. However, these
should be used sparingly (for example, for incidental memory usage during testing). Most of the kernel memory management you will be doing using the
`slab_allocators`. A `slab_allocator`, as used in Solaris and Linux, works like
a cache for memory chunks of a particular size. This reduces both the loss of
memory through fragmentation and the overhead from manipulating the heap
directly. Refer to `include/kernel/mm/slab.h` for the slab allocation and freeing functions you will be using.

## 3.3 Boot Sequence
Most of the boot sequence is handled by the support code. The last thing the
boot loader does is execute the function `kmain()`, which initializes the support
subsystems and then calls `initproc_start()`.
At this point, we are running in the idle process, which is created with the call to `proc_idleproc_init()` right before `kmain()` is executed. We are still in the boot sequence here, which means that we do not
have a thread context in which to properly execute (note that `curthr` is set to NULL in `proc_idleproc_init()`). We cannot block, and we cannot execute user code. The idle proc is a special process that is core-specific - this means that it won't be on the thread list and is only used in `core_switch()` when switching the current thread and process. Feel free to take a look at `core_switch()` in `kernel/proc/sched.c` to understand this better. 

The goal of `initproc_start()` is to set up the first kernel
thread and process, which together are called the init process, and execute its
main routine, `initproc_run`. This should be a piece of cake once you have implemented threads
and the scheduler.
The init process performs further initialization in `initproc_run`, which is also where you should write all of your test code for this project. When your
operating system is nearly complete, it will execute a binary program in user
mode, but for now, be content to put together your own testing system for
threads and processes in kernel mode.

## 3.4 Processes and Threads

Weenix is capable of running multiple processes and threads. There are a few
important things to note about the Weenix threading model.
* There is no kernel mode preemption in Weenix (all threads run until they
explicitly yield control of the processor). It is possible (but not required)
to implement user mode preemption. Think about why this is much easier
to do.
* Weenix is only running on one processor, so synchronization primitives
don’t require atomic compare-and-swap instructions or memory barriers.
* Each process only has one thread associated with it, however the threading code is actually structured so that it would be easy to have multiple
threads per process. For example, each process keeps a list of its threads,
even though that list currently never has more than one entry. If a function
seems unnecessary to you, think of it in the context of multiple threads per
process. For instance, when exiting a thread, you must alert the process that one of its threads exited, even though each process should only ever
have one thread.
* Think of a process as a collection of some number of threads and some
metadata. Killing a process is equivalent to killing its threads and vice versa.

The lifecycle of threads and processes is relatively straightforward. When
a process is created, a thread should be added to it and the process should be
made runnable by adding its thread to the run queue. If a process exits and
it still has child processes, it should reassign those children to the init process,
which, after performing some system setup, will sit in a loop collecting orphaned
processes. When a thread attempts to exit the current process, the process must
clean itself up, because each process only has one thread. Once a thread finishes cleaning up its current task, it makes a call to the process code (`proc_thread_exiting()`) to indicate it is exiting so that the process can do any final cleanup on the thread data structure.

Once all its thread has exited, a process can exit and one of its ancestors
(either its parent or the init process) will call do waitpid() and clean it up fully.
The reason for this somewhat odd deallocation system is that some elements
of the process and thread data structures can only be cleaned up from another
thread’s context. During this assignment, you should determine what these
items are, and clean up everything else in the process as quickly as possible. Also
note that processes which are children of the idle process will not be cleaned up
using this method; they are dealt with separately.

As a side note, you will be using linked lists extensively throughout Weenix.
By this point, you may have looked at the code and seen that some data structures contain list objects or link objects. We provide you with a circular doubly linked list implementation where the links are stored inside of the objects that
are in the list (hence the link fields inside the objects). Most of this list implementation is provided as a set of inline functions which can be found in the `kernel/util/list.c` and a set of macros in `kernel/util/list.h`. There is also a list.py file, which allows you to print and inspect the lists in gdb, which will be very useful for debugging.

The trickiest parts of this segment are do `waitpid()` and `proc_kill_all()`,
not because they are conceptually difficult, but because it is very easy to accidentally introduce bugs that you will discover much later. As such, you should
test the edge cases of these functions early and often throughout the development of your operating system.

## 3.5 Scheduler
Once you have created processes and threads, you will need a way to run these
threads and switch between them. In most operating systems, you must worry
about thread priorities or quality of service assurances, which requires wait
queue as well as run queue optimizations, but the scheduler for Weenix consists only of first-in-first-out queues, a function to switch between the contexts of
two threads, and a few higher-level functions to abstract away the details of the
queues from the caller. There is also a `core_switch()` function which handles the actual
updating of curproc and curthr - this is implemented for you, but it's important to
understand at least the basics of what it's doing in order to implement `sched_switch()`
correctly.

In particular, there is a single run queue from which threads are dequeued
(only when the running thread yields control explicitly) and switched onto the
processor. There are also many wait queues in the system, most of which will
be used as a part of some mutex. When a thread reaches the front of its wait
queue, it can be woken up, which involves putting it on the run queue, waiting
for it to reach the head of the run queue, and then being switched onto the
processor by some other thread running the switching routine.

Switching between threads is the most tricky code in this area, but again, this part
is implemented in `core_switch()`. It takes the first thread off of the run queue, 
sets the global variables for current thread and current process to
point at the new thread and its process, then switches into the new context, all
with interrupts blocked. If the run queue is empty, it calls `load_balance()` to 
attempt to "steal" a thread from another core. However, this isn't relevant for 
you, because Weenix will not be making use of multiple cores. 

## 3.6 Synchronization Primitives

Since the kernel is multi-threaded, we need some way to ensure that certain
critical paths are not executed simultaneously by multiple threads. Once your
scheduling functions work, you should be able to implement synchronization
primitives as well. We make use of both mutexes and spinlocks in Weenix, 
whose implementations you can find in `kernel/proc/kmutex.c` and 
`kernel/proc/spinlock.c`. For this project, you won't need to use mutexes
and only need to use spinlocks if you want to implement multi-core safety.
The mutexes will be used later, in the Drivers and S5FS projects.


**Note:** for 2021 spring, the implementation of kmutex has been provided in assembly form (proc/kmutex.S). We will release the full C source code after the `uthreads` project.

## 3.7 Testing
It is your responsibility to think of boundary conditions which could potentially
cause problems in your kernel. Test code is an important part of software
development, and if you cannot demonstrate that your kernel works properly,
we will assume that it does not.

As mentioned earlier, you should run all your test code from the init process’s
main function (for lack of a better location), although you can add a new file to
hold your tests. To be in the best shape possible before moving on, you should
be able to test all of the following situations.

* Run several threads and processes concurrently. Devise a way to show
that multiple threads are running and that they are working properly.
* Demonstrate that threads and processes exit cleanly.
* Ensure all the edge cases of do waitpid() work correctly.
* Try exiting your kernel both by running proc kill all() and by allowing
all threads to terminate normally.
* Demonstrate that the synchronization primitives work.
* Create several child processes and force them to terminate out of order,
making sure they are cleaned up properly.

Keep in mind that this is not an exhaustive list, but that you should certainly
be able to demonstrate each of these tests passing by the end of this assignment.