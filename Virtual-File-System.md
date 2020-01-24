# Virtual File System

## 5.1 Introduction
The virtual file system, known as the “VFS” provides a common interface between the operating system kernel and the various file systems. The VFS interface allows one to add many different types of file systems to one’s kernel and access them through the same UNIX-style interface. For instance, here are three examples of writing to a “file”:

    $ cat foo.txt > /home/bar.txt
    $ cat foo.txt > /dev/tty0
    $ cat foo.txt > /proc/123/mem

All of these commands look very similar, but their effect is vastly different. The first command writes the contents of `foo.txt` into a file on disk via the local file system. The second command writes the contents to a terminal via the device file system. The third command writes the contents of `foo.txt` into the address space of process 123 (`/proc` is not supported on Weenix).

Polymorphism is an important design property of VFS. Generic calls to VFS such as `read()` and `write()`, are implemented on a per-file system basis. Before we explain how the VFS works we will address how these “objects” are implemented in C.

## 5.1.1 Constructors
File systems are represented by a special type (a `fs_t` struct) which needs to be initialized according to its specific file system type. Thus for each file system, there is a routine that initializes file system specific fields of the struct. The convention we use in Weenix for these “constructors” is to have a function called `<fsname>_mount()` which takes in a `fs_t` object. Note that the `fs_t` to be initialized is passed in to the function, not returned by the function, allowing us to leave the job of allocating and freeing space for the struct up to the caller. This is pretty standard in C. Additionally, some objects have a corresponding “destructor” `<fsname>_umount()`. Construction does the expected thing with data members, initializing them to some well-defined value. Destruction (if the destructor exists) is necessary to clean up any other data structures that were set up by the construction (such as freeing allocated memory, or reducing the reference count on a vnode).

## 5.1.2 Virtual Functions
Virtual functions (functions which are defined in some “superclass” but may be “overridden” by some subclass specific definition) are implemented in the Weenix VFS via a struct of function pointers. Every file system type has its own function implementing each of the file system operations. Our naming convention for these functions is to prefix the function’s generic name with the file system type, so for example the `read()` function for the `s5fs` file system would be called `s5fs_read()`. Every file system type has a struct of type `fs_ops_t` which lists all of the operations which can be performed on that file system. In the constructor for the file system, pointers to these `fs_ops_t` are added to the `fs_t` struct being initialized. One can then call these functions through the pointers, and you have instant polymorphism.

## 5.1.3 Overview
This section describes how the VFS structures work together to create the virtual file system.
Each process has a file descriptor table associated with it (the `proc_t` field `p_files`). Elements in the array are pointers to open file objects (`file_t` structs) in a system-wide list of all `file_t` objects that are currently in use by any process. You can think of this as the system file table discussed in the “Operating Systems Design” lectures. Each process’s array is indexed by the file descriptors the process has open. If a file descriptor is not in use, the pointer for that entry is `NULL`. Otherwise, it must point to a valid `file_t`. Note that multiple processes or even different file descriptor entries in the same process can point to the same `file_t` in the system file table. Each `file_t` contains a pointer to an active `vnode_t`. Once again, multiple system file table entries can point to the same `vnode_t`. You can think of the list of all active `vnote_t`s as the active inode table discussed in the “Operating Systems Design” lectures. Through the `vnode_t` function pointers you communicate with the underlying file system to manage the file the vnode represents.
With all of these pointers sitting around it becomes hard to tell when we can clean up our allocated `vnode_ts` and `file_ts`. This is where reference counting comes in.


### Reference Counting and vnode Locking
As discussed in the overview of VFS, there are a lot of pointers to `vnode_ts` and `file_ts`, but we need to make sure that once all of the pointers to a structure disappear, the structure is cleaned up, otherwise we will leak resources! To this end `vnode_t` and `file_t` both have reference counts associated with them, which are distinct but generally follow each other. These reference counts tell Weenix when a structure is no longer in use and should be cleaned up.

Rather than allocating space for these structures directly, the `*get()` functions described below look up the structures in system tables and create new entries if the appropriate ones do not already exist. Similarly, rather than directly cleaning these structures up, Weenix uses the `*put()` functions to decrement reference counts and perform cleanup when necessary. `vput()` takes a pointer to a `vnode_t*` and will set the vnode_t* to point to `NULL` to ensure vnodes are not used after being "put".  Other systems in Weenix use these functions together with the `*ref()` functions to manage reference counts properly.
For every new pointer to a `vnode_t` or a `file_t`, it may be necessary to increment the relevant reference count with the appropriate `*ref()` method if the new pointer will outlive the pointer it was copied from. For example, a process’s current directory pointer outlasts the method in which that pointer is copied from the filesystem, so you must use `vref()` on the `vnode_t` the filesystem gives you to ensure the `vnode_t` won’t be deallocated prematurely.

**VERY Important Note:** Acquiring a vnode via `vget()` automatically locks the vnode to synchronize access. As a consequence of this locking you can only have one `vget` vnode at a time. If you require a second vnode, you must use the `vget_second` and `vput_second` functions. These ensure that if one kernel thread requires vnode A and then vnode B and another kernel thread requires vnode B and vnode A the kernel will NOT deadlock. 


Note that you may have to add to your old **procs** code in order to properly manage reference counts.
Keeping reference counts correct is one of the toughest parts of the virtual file system. In order to make sure it is being done correctly, some sanity checking is done at shutdown time to make sure that all reference counts make sense. If they do not, the kernel will panic, alerting you to bugs in your file system. Below we discuss a few complications with this system.

### Mount Point References
If mounting is implemented then the vnode’s structure contains a field `vn_mount` which points either to the vnode itself or to the root vnode of a file system mounted at this vnode. If the reference count of a vnode were incremented due to this self-reference then the vnode would never be cleaned up because its reference count would always be at least 1 greater than the number of cached data blocks. Therefore the Weenix convention is that the `vn_mount` pointer does not cause the reference count of the vnode it is pointing to to be incremented. This is true even if `vn_mount` is not a self-reference, but instead points to the root of another file system. This behavior is acceptable because the `fs_t` structure always keeps a reference to the root of its file system. Therefore the root of a mounted file system will never be cleaned up.
It is important to note that the `vn_mtpt` pointer in `fs_t` does increment the reference count on the vnode where the file system is mounted. This is because the mount point’s vnode’s `vn_mount` field is what keeps track of which file system is mounted at that vnode. If the vnode where to be cleaned up while a file system is mounted on it Weenix would lose the data in `vn_mount`. By incrementing the reference count on the mount point’s vnode Weenix ensures that the mount point’s vnode will not be cleaned up as long as there is another file system mounted on it.

### Mounting
Before a file can be accessed, the file system containing the file must be “mounted” (a scary-sounding term for setting a couple of pointers). In standard UNIX, a superuser can use the system call `mount()` to do this. In your Weenix there is only one file system, and it will be mounted internally at bootstrap time.
The virtual file system is initialized by a call to `vfs_init()` by the idle process. This in turn calls `mountproc()` to mount the file system of type `VFS_ROOTFS_TYPE`. In the final implementation of Weenix the root file system type will be `s5fs`, but since you have not implemented that yet you will be using `ramfs`, which is an in-memory file system that provides all of the operations of a S5FS except those that deal with pages (also, ramfs files are limited to a single page in size).
The mounted file system is represented by a `fs_t` structure that is dynamically allocated at mount time.
Note that you do not have to handle mounting a file system on top of an existing file system, or deal with mount point issues, but you may implement it for fun if you so desire, see the additional features section.

## 5.1.4 Getting Started
In this assignment, as before, we will be giving you a bunch of header files and method declarations. You will supply most of the implementation. You will be working in the `fs/` module in your source tree. You will be manipulating the kernel data structures for files (`file_t` and `vnode_t`) by writing much of UNIX’s system call interface. We will be providing you with a simple in-memory file system for testing (`ramfs`). You will also need to write the special files to interact with devices.
Remember to turn the `VFS` project on in `Config.mk` and make clean your project before you try to run your changes. As always, you can run `make nyi` to see which functions must be implemented.
The following is a brief check-list of all the features which you will be adding to Weenix in this assignment.
- Setting up the file system: `fs/vfs.c - fs/vnode.c - fs/file.c`
- The ramfs file system: `fs/ramfs/ramfs.c` (provided in the support code)
- Path name to vnode conversion: `fs/namev.c`
- Opening files: `fs/open.c`
- VFS syscall implementation: `fs/vfs_syscall.c`
Make sure to read `include/fs/vnode.h`, `include/fs/file.h`, and `include/fs/vfs.h`. You will also find some relevant constants in `include/config.h`.
You can wait until the VM project to implement the special file functions which operate on pages.

## 5.2 The ramfs File System
The `ramfs` file system is an extremely simple file system that provides a basic implementation of all the file system operations except those that operate on a page level. There is no need for the page operations until VM, by which point you will have implemented S5FS. Note that `ramfs` files also cannot exceed one page in size. All of the code for `ramfs` is provided for you.

## 5.3 Pathname to vnode Conversion
At this point you will have a file system mounted, but still no way to convert a pathname into the vnode associated with the file at that path. This is where the `fs/namev.c` functions come into play.
There are multiple functions which work together to implement all of our name-to-vnode conversion needs. There is a summary in the source code comments. Keeping track of vnode reference counts can be tricky here, make sure you understand exactly when reference counts are being changed. Copious commenting and [[debug|debugging]] statements will serve you well.

## 5.4 Opening Files
These functions allow you to actually open files in a process. When you have open files you should have some sort of protection mechanism so that one process cannot delete a file that another process is using. You can do that either here or in the S5 layer. Although the choice is up to you, it is somewhat easier to do it in the S5 layer and not worry about it here.

## 5.5 System Calls
At this point you have mounted your ramfs, can look up paths in it, and open files. Now you want to write the code that will allow you to interface with your file system from user space. When a user space program makes a call to `read()`, your do `read()` function will eventually be called. Thus you must be vigilant in checking for any and all types of errors that might occur (after all you are dealing with “user” input now) and return the appropriate error code.
Note that for each of these functions, we provide to you a list of error conditions you must handle. Make sure you always check the return values from subroutines. It is Weenix convention that you will handle error conditions by returning `-errno` if there is an error. Any return value less than zero is assumed to be an error. Do not set the current thread’s `errno` variable or return `-1` in the VFS code, although these behaviors will be presented to the user. You should read corresponding system call man pages for hints on the implementation of the system call functions. This may look like a lot of functions, but once you write one system call, the rest will seem much easier. Pay attention to the comments, and use debug statements. For more information on error handling, see the S5FS Error Handling section.

## 5.6 Testing
You should have lots of good test code. What does this mean? It means that you should be able to demonstrate fairly clearly – via TTY I/O and debug statements – that you have successfully implemented as much as possible without having S5FS yet. Moreover, you want to be sure that your reference counts are correct (when your Weenix shuts down, `vfs_shutdown()` will check reference counts and panic if something is wrong). Note that you must make sure you are actually shutting down cleanly (i.e. see the “Weenix halted cleanly” message), otherwise this check might not have happened successfully. Basically, convince yourself that this works and get ready for the next assignment – implementing the real file system.
We have written some tests for you which you can run from your main process by calling `vfstest_main`, which is defined in `test/vfstest/vfstest.c`. Be sure to add a function prototype for vfstest main at the top of the source file where you call the function. Note that the existence of vfstest is not an excuse for not writing your own tests.