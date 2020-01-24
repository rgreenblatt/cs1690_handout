# Drivers
## 4.1 Introduction
Now that you have processes and threads working properly, you can begin writing the device drivers for terminals, disks, and the memory devices `/dev/zero` and `/dev/null`. The code you will write for this part is in the drivers/ directory.
There are two different types of devices: block devices and character devices. The disk devices you will be writing are block devices, while the terminals and memory devices are character devices. These are similar because they share the
same interface, but there are significant differences in their implementation. In order to make the relationships between devices easier to manage we use some magic.

Remember to turn the `DRIVERS` project on in `Config.mk` and `make clean` your project before you try to run your changes.

Here's a quick introduction to the code structure of this assignment:

- Files you need to modify
    - `drivers/disk/sata.c` - a block device that handles read and write to disks
    - `drivers/tty/ldisc.c` - line discipline that handles interaction with terminals
    - `drivers/tty/tty.c` - tty implementation
    - `drivers/memdevs.c` - memory devices (`/dev/zero` and `/dev/null`)
- Other files (you shouldn't have to modify these)
    - `drivers/tty/vterminal.c` - virtual terminal implementation
    - `drivers/blockdev.c` - block devices
    - `drivers/chardev.c` - character (byte) devices
    - `drivers/keyboard.c` - implementation of keyboard interaction
    - `drivers/pcie.c` - PCIe
    - `drivers/screen.c` - code that actually writes stuff to the screen

## 4.2 Object-Oriented C

As you look through the struct definitions of the different devices in the header files, you should notice that the structs for more specific types of devices (e.g. ttys and disks, referred to as the sub-struct from here on) contain the structures for the generic devices (e.g. block and character devices, referred to as the superstruct) as fields. These are not pointers; they occupy memory that is part of the sub-struct. This way, we can use memory offsets to cast both from substruct to super-struct, and vice versa. So, given a pointer to the struct of a char device which you know is a terminal device, just subtract the size of the rest of the terminal device struct and you have a pointer to the beginning of the terminal device struct. There is a macro provided for the purpose of doing these pointer conversions called `CONTAINER_OF`. In many cases, a more specific macro is defined using `CONTAINER_OF` which converts from a super-struct to a specific sub-struct (for an example, see `bd_to_tty` in `drivers/tty/tty.c`).

You should also notice that one of the fields in the device structs is a pointer to a struct containing only function pointers. The generic device types (e.g. block and character devices) each specify that they expect certain operations to be available (e.g. `read()` and `write()`). This function pointer struct contains pointers to the functions which implement the expected operations so that we can perform the correct type of operation without actually knowing what type of device struct we are working with. The definitions of these function pointer structs are in the C source files of their respective types. Essentially, we are manually implementing a simple virtual function table (which is how C++ implements polymorphism).

## 4.3 TTY Device
Each tty device is represented by a `tty_t` struct. A tty consists of a driver and a line discipline. The driver is what interfaces directly with the hardware (keyboard and screen), while the line discipline interfaces with the user (by buffering and formatting their tty I/O). In Weenix, the main purpose of the tty subsystem is to glue together the low level driver and the high level line discipline. The advantage of this is that when a program wants to perform an I/O operation on a tty, it does not need to know the specifics of the hardware or the line discipline. All of these implementation details are abstracted away and dealt with by the functions in `drivers/tty/tty.c`.

Once you have a working virtual file system (after VFS) you will access the terminals through files, specifically in the files `/dev/tty0`, `/dev/tty1`, etc. Then you can read and write to the terminals using the `do_read` and `do_write` functions. Until then, you will need to use the `chardev_lookup` function to get devices and then explicitly call the device’s read/write functions in order to test your code. A convenient way to do this is by using the kernel shell. For more details, see the testing section at the end of this chapter.

### 4.3.1 Line Discipline
The line discipline is the high-level part of the tty. It provides the terminal semantics you are used to. Essentially, there are two things the line discipline is responsible for – buffering input it receives and telling the tty what characters to print. Buffering input is what allows users to edit a line in a terminal before pressing enter, or, as we call it, cooking the buffer. The buffer for a line discipline is split into two sections, raw and cooked, so that the buffer can be filled circularly. Before a circularly-contiguous segment of the buffer is cooked, the user is able to edit it by editing the current line of text on the screen. When the user presses enter, that segment of the buffer is cooked, a newline is echoed, and the text becomes available to the running program via the read system call. This is why the read system does not return until it receives a newline when reading from a terminal. For simplicity, do not store more input than you can put in the primary buffer (even though you could theoretically use the buffer that the waiting program provided as well).

The other important job of the line discipline is telling the tty which characters to echo. After all, when you type into a terminal, the characters you press appear on the screen. From the user’s perspective, this all happens automatically. From your perspective (you being a kernel hacker), this must be done manually from the tty and the line discipline. The line discipline is also responsible for performing any required processing on characters which will be output to the tty via the write system call. In Weenix, the only characters that are not just echoed are the newline, backspace, and `Ctrl+D` characters.


### 4.3.2 TTY Driver
The tty driver is the low level part of the tty, and is responsible for communicating with hardware. When a key is pressed, the appropriate tty driver is notified.
If a handler was registered with the driver via the `intr_handler` function, the key press is sent off to the handler. In our case, the tty subsystem registers the `keyboard_intr_handler` function with the driver, which calls the line discipline’s `tty_receive_char_multiplexer` method. Then, after any high level processing, any characters which need to be displayed on the screen are sent directly to the driver via the `vterminal_write` function.

The tty driver is already implemented for you. For anyone feeling adventurous, feel free to take a look at `drivers/keyboard.c`, `drivers/screen.c`,
and `drivers/tty/vterminal.c`.

## 4.4 Disk Device Driver
A block device is associated with each disk device. This structure contains information about what threads are waiting to perform I/O operations on the disk, any necessary synchronization primitives, the device ID, and any other information required by the SATA protocol. In Weenix, you can assume that all disk blocks are page-sized. We have defined the `BLOCK_SIZE` macro for you to be the same size as the size of a page of memory; use it instead of a hard-coded value.

## 4.5 Memory Devices
You will also be writing two memory devices, a data sink and source. These will not really be necessary until VFS. Still, these fit will with the other device drivers so they are included in this part of the assignment. If you have played around with a Linux/UNIX machine, you might be familiar with `/dev/null` and `/dev/zero`. These are both files that represent memory devices. Writing to `/dev/null` will always succeed (the data is discarded), while reading from it will return `EOF` immediately. Writing to `/dev/zero` is the same as writing to `/dev/null`, but reading any amount from `/dev/zero` will always return as many zero bytes (’\0’) as you tried to read. The former is a data sink and the later is a data source. The low level drivers for both of these will
be implemented in `drivers/memdevs.c`.

## 4.6 Testing

As always, it is important to stress test your terminal code.

* Have two threads read from the same terminal, which will cause each thread to read every other line. If you’re not sure why this is, ask.
* Make sure that, if the internal terminal buffer is full, Weenix cleanly discards any excess data that comes in.
* Ensure that you can have two threads simultaneously writing to the same terminal.
* Stress test your disk code. This will not be needed until the S5FS assignment, but it is a good idea to make sure it works now.
* Make sure that you can have multiple threads reading, writing, and verifying data from multiple disk blocks.
* Think of ways that you can write to a disk and display the data stored there.
* Note that Weenix does not currently support Caps Lock.

It is very important that you get this code working flawlessly, since it can be a constant source of headaches later on if you don’t. Note that the disk driver only works with page-aligned data, so you should use `page_alloc()` to allocate the memory used in your test cases, not `kmalloc()`.

Since you should now have a functional tty, you should try using the kernel shell to test it out. Once you are confident in your tty code, try implementing your own `kshell` commands to run further kernel tests. Below is what to add to initproc run (`#include "test/kshell/kshell.h"` at the top of `kernel/main/kmain.c`):

```
/* Add some commands to the shell */
kshell_add_command("test1", test1, "tests something...");  
kshell_add_command("test2", test2, "tests something else...");

/* Create a kshell on a tty */
int err = 0;
kshell_t *ksh = kshell_create(ttyid);
KASSERT(ksh && "did not create a kernel shell as expected");

/* Run kshell commands until user exits */
while ((err = kshell_execute_next(ksh)) > 0);
kshell_destroy(ksh);  
```
