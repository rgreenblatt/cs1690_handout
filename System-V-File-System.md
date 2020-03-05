Under construction
<!--System V File System 
====================

Introduction
------------

The System V File System, or “S5FS,” is a file system based on the original Unix file system. Although it lacks some of the complex features of a modern file system, Weenix uses it because it provides an excellent introduction to the issues involved in writing an on-disk file system.

In completing the S5FS assignment, you will provide its implementation for the full VFS interface. You will come across many different S5FS objects that interact with each other. Most of these object types are on-disk data structures – that is, the memory they occupy is actually being saved to disk (the only S5FS type which is not backed by disk is the struct representing the file system itself).

Remember to turn the `S5FS` project on in `Config.mk` and `make clean` your project before you try to run your changes.

Disk Layout
-----------
<p align=center>
[[figures/s5fs/layout.svg]]
<p align=center><sup>The default disk layout for Weenix.</sup></p>
</p>

### Superblock

The first block of the disk (block number zero) is called the superblock, which contains metadata about the file system. The important data fields inside it are the inode number of the root directory, the number of the first free inode, the first section of the free block list, the number of free blocks currently referenced in that section of the free block list, and the total number of inodes in use by the filesystem. The less important fields are the “magic number” for S5FS disks (used as a sanity check for the OS to determine that the disk you are reading is formatted as an S5FS disk), and the version number of S5FS we are using. The in-memory copy of the superblock is stored in a `s5_super_t` struct. For more information about the structure of the free block list, check out the section on below.

### Inodes

Next, there is an array containing all the inodes for the file system. Each inode requires 128 bytes of space, and each disk block is 4096 bytes, so we store 32 inodes per block. Inodes are referred to by their index in the array, and are stored in memory as `s5_inode_t` structs. Each inode is either free, or it corresponds to some file in the file system.

If an inode is not free, it represents a file presently on disk. The inode holds of the size of the file, the type of the file (whether it is a directory, character device, block device, or a regular file), the number of links to the file from other locations in the file system, and where the data blocks of the file are stored on disk.

If an inode is free, it need only be marked as empty and contain the inode number of the next free inode in the free list (or `-1` if it is the last element in the list). This link is stored in the `s5_freesize` field (you can think of that field as being a `union` whose interpretation depends on whether the inode is free or not).

<p align=center>
[[figures/s5fs/inode_with_indirect_block.svg]]
<p align=center><sup>An node with an indirect block (data blocks not pictured).</sup></p>
</p>

As mentioned above, the location of the data blocks for a file are also stored in the inode for the file. The inode itself keeps track of `S5_NDIRECT_BLOCKS` data block numbers directly, but this is not usually enough for a large-ish file. Luckily, S5FS inodes for “large” files also contain a pointer to an indirect block, which is a disk block filled with more data block numbers (in case it isn’t clear: by “pointer” in this context we mean a disk block number, not a memory address). It is able to store up to `S5_BLOCK_SIZE / sizeof(int)` more block numbers. In a production file system, you should be able to support arbitrarily long files, which would require arbitrarily long indirect block chains (or, more likely, B-trees), but in Weenix we choose to only implement a single indirect block for simplicity. This means that there is a maximum file size; make sure you handle this error case correctly.

While the default disk size gives you space for several hundred data blocks, the indirect block will allow a single file to use thousands of blocks. This might seem unnecessary, however it allows for the implementation of sparse files. If a file has a big chunk of zeros in it, Weenix will not waste actual space on the disk to represent them; it just sets the block index to zero. When reading a file, if a block number of zero is encountered, then that entire block should consist of zeroes. Remember that zero is guaranteed to be an invalid data block number because it is the block number of the superblock.

<p align=center>
[[figures/s5fs/free_block_list.svg]]
<p align=center><sup>The free block list.</sup></p>
</p>

### Data Blocks 

Data blocks are where actual file contents are stored. They occur on disk after the inode array and fill the remainder of the disk. For simplicity, disk blocks and virtual memory pages are the same number of bytes in Weenix, although this is not necessarily true for other operating systems.

The contents of the data blocks are obviously dependent on what file they are filled with (except for directories, which also use data blocks but have a special format described below) unless they are in the free block list. Instead of a free list where each link only points to one more link, which would be wildly inefficient due to frequent seek delays, S5FS uses a list where each link in the list contains the numbers of many free blocks, the last of which points to the next link in the free list. The first segment of the free list is stored in the superblock, where up to `S5_NBLKS_PER_FNODE` blocks are stored. The last element of this array is a pointer to a block containing `S5_NBLKS_PER_FNODE` more free blocks, the last of which is a pointer to a block with more free pointers, and so on. The last free block in the list has a `-1` in the last position to indicate there are no more free blocks. After the second-to-last free block in the superblock’s array is used, the next set of free blocks should be copied out of the next block, and then the block they were just copied from can be returned as the next free page.

### Directories

S5FS implements directories as normal files that have a special format for their data. The data stored in directory files is essentially just a big array of pairs of inode numbers and the filenames corresponding to those inode numbers. Filenames in S5FS are null-terminated strings of length less than or equal to `S5_NAME_LEN` (including the null character). Any entry with a zero-length name indicates an empty or deleted entry. Note that every directory contains one entry for “`.`” and one for “`..`”, corresponding to the current directory and the parent directory, respectively, from the beginning of its existence to the moment it is deleted. The link count for a newly-created directory should be two (one reference from its parent directory, and one from itself).

Caching
-------

At this point, you know a lot about how the on-disk filesystem looks and could probably inspect the disk block-by-block and understand what files are stored there. However, while working on this part of Weenix, you will not need to directly read and write from the disk, even in the most low-level functions. Instead, you will use the VM caching system to read blocks from disk into memory. You can then manipulate these pages in memory, and the Weenix shutdown sequence will automatically handle writing them back to disk.

The Weenix caching system uses two different types of objects: page frames, which are each responsible for tracking one page/block of data, and memory objects, which are each associated with a number of page frames that hold the data for that memory object. In the codebase, the names for these objects are `pframe_t`s and `mobj_t`s, respectively. Each memory object represents some data source, which could be a file, device, or virtual memory region. This means that page frames are used to reference the blocks of files, blocks of a device, and blocks of segments of memory. Specifically, page frames store some metadata about the page they hold and a reference to that page in memory. If a particular page of, say, a file hasn’t been paged into memory, there will not yet be a page frame for that page.

In general, to get a particular page frame from a memory object, you should call the `mobj_get_pframe()` function on the memory object you wish to get the page from. The data stored in the page is stored at the location pointed to by the page frame’s `pf_addr` field. If the call to `mobj_get_pframe()` requests a page frame for writing to, the returned page frame will be marked so that it will be cleaned (the changes will be written back to disk) later. The cleaning process uses callbacks in the disk’s memory object to write the data back to disk. Importantly, `mobj_get_pframe()` returns with the requested page frame's mutex locked so don't forget to call `pframe_release()` once you're finished using it to unlock the page frame again. 

In S5FS, we provide you with `s5_get_disk_block()` , which handles some synchronization and state checking but is effectively a wrapper for `mobj_get_pframe()`. Similarly, `s5_release_disk_block()` will unlock the page frame's mutex.

To be fixed
--------------

To use an inode from disk, you must get its page from the disk memory object (the `S5_INODE_BLOCK()` macro will tell you which disk block to get) and then use the `S5_INODE_OFFSET()` macro to index into the page. When you are changing a file or the filesystem data structures, make sure that you remember to dirty the inode if necessary (mark it as modified so it can be written back to disk). Note the presence of the `dirtied_inode` field in `s5_node_t`, which can be set for this purpose. Remember that you should never clean pages yourself as either the the Weenix shutdown sequence will take care of that automatically.

While working on S5FS, you may notice that there are two very similar methods for accessing the data on disk: calling the `get_pframe()` operation of the memory object for the block device (the disk) and on the memory object for the file. Therefore, it can sometimes be confusing which one to use. Although this may sound like common sense, it is important that you use a file’s memory object every time you are dealing with a file, and use the block device’s memory object when you are implementing pieces of the filesystem that are “low-level enough” to know how a file is split across multiple blocks on disk. If you accidentally use the disk memory object instead of the file memory object, you will be short-circuiting a couple layers of abstraction that will be necessary later on.

Error Handling 
--------------

You always need to check for things that can go wrong. When an error condition occurs, you should return `-errno` where `errno` is the error number that indicates the type of error that occurred. For example, if you run out of free data blocks when attempting to write to a new block of a file, you should return `-ENOSPC`. You should *always* check the return value from non-void functions you call, and if the returned value is negative you often need to propagate it up. Returning `-1` or setting the current thread’s `-errno` variable is not correct.

However, it is also important that your VFS code check for as many errors as possible, so that each file system that it runs need not check the same error cases. If you know that some error condition should always be dealt with at the VFS layer, you should assert that it does not occur in the S5FS layer to identify bugs in your implementation while you develop.

We have tried to indicate some errors that you should check for in the comments, but we have probably not mentioned all of them, so you should go over your code thoroughly to make sure that you handle all possible errors.

Getting Started
---------------

You need to implement:

-   The high-level system calls for S5FS (the VFS interface). Many of these will have a name such as `s5fs_[name]` with a corresponding VFS call named `do_[name]`.

-   The low-level subroutines. These subroutines represent common functionality that may be useful to reuse in many of the high-level system calls.

-   A few memory management routines (such as `pframe_get()`). This is to understand a little better how the caching system works.

Testing
-------

Your test cases should demonstrate clearly that all functions have been tested properly. While much of the functionality of S5FS will be tested by the tests you used for VFS, there are several cases that may require a bit more thought:

-   Indirect blocks. (The `hamlet` file on your virtual disk is provided as an example of a file which needs an indirect block, but don’t forget to check the case where you need to allocate one at runtime as well.)

-   Sparse blocks.

-   Running out of inodes, data blocks, or file length.

-   Shutting down and rebooting does not erase your changes.

We have written some tests for you which you can run from your main process by calling `s5fstest_main`, which is defined in `test/s5fstest.c`. Note that the existence of these tests is *not* an excuse for not writing your own tests.

Be sure that your link counts are correct (`calculate_refcounts()` will calculate the counts for you). Note that you *must* make sure you are actually shutting down cleanly (i.e. see the “Weenix halted” message). Reference count issues will prevent Weenix from shutting down cleanly.

To ease the difficulty of debugging your file system code, we have provided a couple of utilities to help you develop. The `fsmaker` utility will come in handy for inspecting blocks, inodes, and other data structures on your virtual machine’s disk. To read more about how to use `fsmaker`, run `fsmaker –help` from the root of your development directory. Your disks are stored in files on the host operating system (the `user/disk*.img` files), and must be passed to `fsmaker` as an argument. Also, running the `./weenix` script with the `-n` option will create a brand new disk for you and fill it with a bunch of sample files and directories. To begin this assignment, you must use this option at least once, otherwise you will not have a disk to work with.
-->