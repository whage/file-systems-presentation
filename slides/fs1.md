# File systems

### related concepts
devices -> disks -> partitions -> file systems

---

- "device special files" or "device nodes" (devices)
- a device special file corresponds to a device on the system
- there are 2 main types of **devices**:
    1. **character devices**: handle data character-by-character: keyboards, terminals, etc... ["c" at `ls -l`]
    2. **block devices**: handle data a block at a time, typically `n*512b` ["b" at `ls -l`]
    This size is the smallest unit of disk access

---

- each device has a corresponding **device driver** within the kernel
    - device drivers implement a set of operations that correspond to I/O actions
- some devices are real (mouse, disk, tape drive), others are virtual, meaning there is no corresponding hardware

---

- device files appear within the file system, usually under `/dev`
- device files have a major ID and a minor ID (both recorded in the i-node for the device file)
- hard disks:
    - major ID: identifies the type of the device and thus the device driver of the kernel associated with that type [1st number at `ls -l`]
    - minor ID: uniquely identifies a particular device, it is passed to the driver [2nd number at `ls -l`]

(https://www.oreilly.com/library/view/linux-device-drivers/0596000081/ch03s02.html)

---

## Disks - TODO: SSDs
An important type of devices: **hard disks**

- mechanical devices
- spinning platters
- tracks / sectors / blocks
- seek time

---

![Hard disk](slides/hard-disk-structure.png)

---

- each disk is divided into one or more nonoverlapping **partitions** (logical disks)
- each partition is treated by the kernel as a separate device under /dev
    - /dev/sda is the first hard drive
    - /dev/sda1 is the first partition of the first hard drive

---

![Partitions](slides/partitions.png)

---

- a partition may hold any type of information, but usually one of the folowing:
    - a file system holding regular files and directories
    - a data area accessed as a raw-mode device
    - a swap area used by the kernel for memory management

---

- advantages of having multiple partitions:
    - have multiple OSes
    - encapsulate data, because filesystem corruption is local to a partition
    - prevent runaway processes from consuming all the sapre space
    - increase disk space efficiency: different partitions can have different block sizes (small files with big block size is a waste)
    - files are stored closely together: lower seek times

---

- "volumes": (https://unix.stackexchange.com/questions/87300/differences-between-volume-partition-and-drive)
    - The term volume in Linux is related to the Logical Volume Manager (LVM), which can be used to manage mass storage devices
    - A physical volume is a storage device or partition. A logical volume created by the LVM is a logical storage device which can span multiple physical volumes.

---

### File Systems

"file system" - two different meanings:
1. the entire hierarchy of directories
2. the type of the file system: how data is organized on a computer disk (on the byte level)

(http://www.linfo.org/filesystem.html)

---

- a file system provides **3 key abstractions** over the pyhsical medium on which the data is stored:
    1. file: makes it seem like the data is one single contiguous string of bytes
    2. file name: instead of having to remember the physical location of the pieces, we get to use a name that identifies the data
    3. directory tree: a hierarchical system to help organize the data (as it was in the old times on the shelves)
- so a file system is an organized collection of regular files and directories

---

- Linux supports a wide variety of file systems
    - `cat /proc/filesystems` to see fs types currently know by the kernel

---

- "partitioning" vs "formatting":
    - partitioning: creating logical units of space on a drive, it contains a single type of file system
    - formatting: the process of creating a filesystem (usually on a partition of a storage device)

(https://superuser.com/questions/1162758/what-is-the-difference-between-partitioning-and-formatting-a-disk)

---

- the basic unit for allocating space is a **logical block** which is some multiple of contiguous physical blocks

---

layout of a partition with a generic filesystem:

![Partitions](slides/partitions.png)

---

- boot block is not used by the filesystem, it contains info to boot the OS, all fs-s have a boot block, but most are unused
- the superblock contains
    - the size of the i-node table
    - the size of logical blocks
    - size of the fs in logical blocks
- each file and dir has a unique entry in the i-node table, this entry records various information about the file
- data blocks: the great majority of space in afile system

---

- different fs-s residing on the same physical device can be of different types and sizes! (different parameter settings)
- note the difference between "a concrete file system (with a UUID)" and "file system type"

---

### I-nodes (index-node)

- the fs's i-node table contains one i-node for each file and directory residing in the file system
- an inode contains metadata about a file
- **i-node number** (`$ ls -li`) - their sequential location in the i-node table

---

- information maintained in an i-node:
    - file type
    - owner
    - group
    - access permissions for three categories
    - 3 timestamps (...)
    - number of hard links to the file
    - size of the file in bytes
    - number of blocks allocated to the file
    - pointers to the data blocks of the file

---

- ext2 (and most other filesystems) doesn't store the data blocks of a file contiguously. This way it can reduce
the fragmentation of free disk space: the wastage created by the existence of numerouse pieces of noncontiguous free space, all of which are too small to use.

---

- under ext2, each i-node contains 15 pointers - see p.257 in the book, interesting stuff!

---

![Partitions](slides/inode.png)

---

- To the Linux kernel the filesystem is flat. That is, it does not
    - have a hierarchical structure
    - differentiate between directories, files or programs or
    - identify files by names. Instead, the kernel uses inodes to represent each file

(http://www.linfo.org/filesystem.html)

---

### The VFS - Virtual File System
- a kernel feature that creates an **abstraction layer** for file-system operations

---

![Partitions](slides/vfs.png)

---

- defines a generic interface for file-system operations, programs specify their fs operations in terms of this interface
    - open(), close(), read(), write(), lseek(), truncate(), stat(), mount(), unlink(), symlink(), etc...
- each file system provides an implementation for the VFS interface
- allows client applications to access different types of concrete file systems transparently

---

### Single Directory Hierarchy and Mount Points (p.261)
- all files from all file systems reside under a single directory tree even if they are stored on a different disk or computer
- fun fact: Filesystem Hierarchy Standard (FHS) defines the main directories and their contents for Unix-like OSes

---

- other file systems are **mounted** under the root directory and appear as subtrees within the overall hierarchy
- when we mount, we specify a **device** that contains the file system we want to mount
- mounted == connected/attached to the file system hierarchy

---

- `$ mount <device> <directory>`
    - attaches the file system contained on the named device into the directory hierarchy at the specified **directory - the mount point**
- `$ umount <device/mount_point/partition_name>` - unmount the filesystem identified by `<device/mount_point/partition_name>`
- `/etc/fstab`: the file system mount table - specifies the filesystems to be mounted at boot time

---

### Journaling file systems
- Updating file systems to reflect changes to files and directories usually requires many separate write operations

---

- interruption (power failure or system crash) -> data structures left in an invalid intermediate state
    - for example, deleting a file on a Unix file system involves three steps:
        - Removing its directory entry
        - Releasing the inode to the pool of free inodes
        - Returning all disk blocks to the pool of free disk blocks
- **without journaling, a full file system check is required** which can take hours!

---

- a special file, called a **journal** is used to keep track of the data that is to be written to the disk
- using journaling, it is enough to look at the journal to reconstruct the consistent state
- writes are stored in **transactions**; if the transaction finishes writing to disk, its data in the journal is committed to the filesystem
- the file being worked on may still be lost, but the filesystem itself remains consistent

---

### FUSE (since v2.6.14)
- traditionally, file systems were implemented as part of the kernel
- FUSE is a framework/mechanism that adds hooks to the kernel that allow a file system to be completely implemented via a user-space program,
without needing to patch or recompile the kernel

---

- create file systems with custom functionality without touching the kernel
    - many libraries and languages are available
    - user-space bugs' impact is more contained

"To FUSE or Not to FUSE: Performance of User-Space File Systems"
(https://www.usenix.org/system/files/conference/fast17/fast17-vangoor.pdf)
