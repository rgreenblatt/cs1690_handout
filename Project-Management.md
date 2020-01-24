# Project Management
## 2.1 Introduction
The first section of this chapter guides the reader through the process of producing a Weenix assignment from the project’s complete source code (useful
for course instructors), and the second explains how to set up a development
environment for implementing the assignment yourself (useful for students).
## 2.2 Preparing Weenix Assignments
This section will walk you through a sample run of Weenix, and then shows you
how to remove sections of code from the full implementation to allow you or
your students to re-implement them.
### 2.2.1 Quick Start: Getting Weenix Running
See instructions in the top-level README file.
### 2.2.2 Getting the Source
Requests for a copy of Weenix may be made via email to weenix-devel@lists.cs.brown.edu.
### 2.2.3 Dependencies
The following is required to run Weenix:
* gcc, the GNU C compiler
* make, the GNU build system
* qemu,the processor emulator that Weenix runs on top of
* xorisso and grub-mkrescue (necessary to generate the disk image used
by the emulator)
* python, at least 2.6 (but 2.x only)
* bash

The following is optional, but recommended:
* git, the version control system
* gdb, the GNU debugger, and xterm
* cscope (for easier browsing of the Weenix source)

### 2.2.4   Configuration
Configuration for Weenix is available through editing the files Config.mk and
Global.mk, which contain VARIABLE=value assignments in make syntax. Global.mk
contains directives relevant to the environment that Weenix is running in (e.g.
alternative locations for utilities like python or gdb), while Config.mk is used
to configure values that affect the behavior of Weenix itself (such as what modules to enable during the compilation process, or the size of the virtual disk to
create).
In particular, you should be aware of the directives that affect what assignment to enable. These are the first directives of Config.mk. For example, to
enable the S5FS assignment, set the first three to 1 (DRIVERS=1, VFS=1, and
S5FS=1).

### 2.2.5   Blanking Solutions
This section will teach you how to blank the solution for individual assignments.
You will wish to do this before you attempt to complete the Weenix assignments
(otherwise, you will find that they have already been done for you!) or before
you assign them to others.

The script make-weenix-repo.py in the tools/ directory performs this task.
It removes the contents of specially-marked functions, putting a NOT YET IMPLEMENTED()
marker in their place. It also initializes a fresh git repository in the root of the
source tree for you (or your students) to clone from.

1. Obtain an unmodified copy of the Weenix source tree.
If you skip this step, you will lose all of your work: the make-weenix-repo.py
will obliterate any git history that exists in the current source tree, as well
as the contents of assignments that are to be removed.
2. Change into the tools/ directory.

3. Run ./make-weenix-repo.py --cutextra all --cutsource [ASSIGNMENTS],
where [ASSIGNMENTS] is a comma-delimited list of the projects you
want to assign (or “all”, without quotes).

For example, suppose you want to implement the S5FS and VM assignments. You should blank the solutions for these assignments, and leave the rest (PROCS, DRIVERS, and VFS) intact. This can be accomplished
with the command:

`$ ./make-weenix-repo.py --cutextra all --cutsource S5FS,VM`

The order of the assignments is: PROCS, DRIVERS, VFS, S5FS, and
lastly VM. Remember that each assignment depends on its predecessor in
order to run. For example, if you wish to start with the VM assignment,
do not blank any other assignments, or you will find yourself unable to
run the VM code you write.

As another example, suppose you want start fresh from the first assignment. Run:

`$ ./make-weenix-repo.py --cutextra all --cutsource all`

4. Update the assignment directives in Config.mk to reflect the first assignment you will work on.
These are described in detail in the Configuration section.
For example, if you are starting with the VFS assignment, set your assignment directives to:

    DRIVERS=1

    VFS=1

    S5FS=0

    VM=0

    DYNAMIC=0

If you are making use of the git repository, you will need to commit this
change. For example:

`$ git commit -m ’Updated Config.mk assignments directives’ Config.mk`

Congratulations: you are now ready to start the assignment. If you need to
distribute the assignment to others, have them copy (or git clone, if you are
using git) this source tree to be their fresh copy to start working on.

## 2.3 Implementing Weenix Assignments
This section introduces some essential concepts in Weenix to e.g. a student who
is interested in completing a Weenix assignment.

### 2.3.1 A Message to the Reader
Although this section is ostensibly about getting to know the Weenix codebase,
we realize this is also the last part of the book we can be certain everyone will
read, so we would like to give a few pieces of advice.
* It is possible that this is your first exposure to a large code base. You
should definitely spend some time poring over the existing code, thinking
about what to implement. Take a look in Appendix B to learn more about
working in large codebases.
* For this and future assignments, it may be helpful to draw out a call graph
(which functions call which) for the code you will be writing. Taking notes
about the code as you read it is a useful skill for any codebase.
* Become an expert at using your chosen development tools. This will save
you countless hours of time and energy, whether you learn to use a debugger (see Appendix C), text editor/IDE, cscope, version control, or
something else. Many of these are mentioned in the appendix, so take a
look there for some tips for getting started.
* Be sure to ask questions! The course staff is friendly and has a great
working knowledge of the OS because they have implemented it themselves. Beyond that, getting Weenix working is far more rewarding if you
understand why something is done a particular way, and the repercussions
of choosing a different way to do something.
* Take breaks to relax. :-)

### 2.3.2 Build System
The Weenix build system is based on the popular Unix utility make. It is
meant to aid development by automating the compilation and linking of binaries,
although it is used to automate several other things in the Weenix codebase as
well.

In order to build the full operating system, make must be run in the top
directory of the Weenix repository. Each make target corresponds to particular
a set of commands listed in the Makefile located in the current directory. In the
case of the top-level Makefile, make all simply descends into the kernel and
user directories and executes make within each. The kernel and the userspace
binaries can be built separately, by calling make from their respective directories.
There are many different targets, such as individual object (.o) files, linked
binaries and disk images, but the most useful targets are listed below.
To compile all the code using eight concurrent tasks, a user should run:

`$ make -j 8 [all]`

or potentially just:

`$ make all`

`$ make % defaults to target "all"`

if the pretty output is more important than the speed of compilation.
Sometimes, such as after a change in the configuration settings or compiler
flags, it is necessary to compile all the source code from scratch. However, make
will only rebuild a file if it has been modified since the last time it was built.

To delete all the constructed binaries and force make to compile from scratch, a
user may run:

`$ make clean; make all`

### 2.3.3 Assignment Specification
To find out what to implement for a newly opened assignment, use the following
command:

`$ make nyi`

This will produce a list of all the functions that are not yet written in the
codebase with labels telling what file they are in and what assignment they are
associated with.

Note that this command is only to be used with blanked assignments (otherwise, there’s nothing to implement!).

### 2.3.4 Running
The weenix script has been provided for running the operating system inside a
virtual machine or hardware simulator. It should be invoked from the topmost
directory of the Weenix repository. To run it, one may simply use the command:

`$ ./weenix`

Until there is a functional on-disk file system, this is all that is necessary to
run Weenix. However, at that point, the ability to create disks becomes very
important. It is necessary to create a fresh disk the first time S5FS runs, and
also after any time the disk is corrupted by a buggy kernel or unclean shutdown.
The following command will run Weenix with a newly constructed disk:

`$ ./weenix -n`

At some point, it may be helpful to use a debugger to aid in the development
of the operating system. The following command will begin Weenix in a virtual
machine and attach gdb to it (if the virtual machine in use supports this option).

`$ ./weenix -d gdb`

The QEMU monitor can produce additional useful debugging information
from the virtual machine. The QEMU monitor will be displayed on stdout
instead of your debug messages, allowing you to query the virtual machine
directly. Refer to Appendix C for more on its useful commands.

`$ ./weenix -d qemu`

These options (and their exciting longer names --new-disk and --debug)
are also visible at the command line.

`$ ./weenix -h`

### 2.3.5 Making Changes
Any time someone works on a large enough project, they will inevitably make
difficult-to-reverse mistakes or accidentally delete their work. The best way to
prepare oneself for this inevitability is to use a version control system. The
Weenix repository uses the source control system git, and everyone who undertakes a project of this magnitude should become at least competent enough
to make commits, revert a file back to a previous commit, and make feature
branches for each new addition. The following list of steps are a highly recommended way to implement a new piece of Weenix.
1. Make a new branch to develop in with a descriptive name.
2. Write tests for the new feature, running them frequently.
3. Write code, committing relatively frequently to avoid losing any work.
4. Make sure all the tests pass and that they comprehensively cover the set
of requirements.
5. Push the changes back onto the main branch once everything is functional.
6. Back up the entire project frequently using an external hard drive or (even
better) an online backup and recovery service.
Even if you don’t follow these steps, please, for the love of all things holy,
use source control. If you mess up your repository due to lack of experience,
somebody will be able to help you out of the mess and you’ll be back where
you were before, but if you mess up an unprotected directory, there is no going
back! Having a student who loses all their Weenix code is awful, and being that
student has been a recurring theme in many of our worst nightmares. If you
have any questions or have never used source control before, feel absolutely free
to reach out to the TA staff in setting it up.

