# Kernel Internals 

*In here, we are going to discuss what rootkits actually maipulate*

## Core Concept

The kernel does not store 'processes, files, or network connections' as human concepts.
It stores data structures in memory, and every tool you use (`ps`, `ls`, `netstat`, `top`) is simply a formatted view of those structures.

When you run `ps aux`, you are not 'seeing processes', you are seeing the kernel walking a linked list of structures and printing what it finds.

When you open a file:

`cat /etc/passwd`

You are not accessing a file directly, you are following a chain of pointers that eventually leads to disk blocks.

*Rootkits don't hide things, they break, alter, or redirect the links between kernel structures.* 

If you don't understand those structures, rootkit is like a magic happening out there, once you do, it becomes inevitable.

## What the Kernel actually manages

As its core, the kernel exists to manage and coordinate:

**1.** Execution (processes and threads)
**2.** Memory
**3.** Filesystems
**4.** Networking
**5.** Credentials and privileges
**6.** Hardware access

But internally, all of these are just **objects in memory**, connected by pointers.

The kernel is not a 'program', it is a living graph of structures constantly being traversed, modified, and validated.

## Process Representation (What a "Process" Really is)

A process is not a running binary, it is a structure in kernel memory that describes execution state.

#### Linux: `task_struct`

In Linux, every process is represented by a structure called `task_struct`.

##### Conceptually simplified:

```C
struct task_struct {
    volatile long state;
    pid_t pid;
    struct task_struct *parent;
    struct list_head tasks;   // Linked list of all processes
    struct mm_struct *mm;     // Memory descriptor
    struct files_struct *files;
    struct cred *cred;        // Credentials
    ...
};
```
This structure does not "contain" the process, it describes the process.

The most important field here is:

`struct list_head tasks;`

This is how the kernel knows which processes exist, all processes are linked together in a doubly-linked circular list:

`init → bash → sshd → apache → init`

Each process points to the next and previous, when tools like `ps` run, they do not 'scan memory', they walk this linked list.

### How process hiding works conceptually

If a rootkit removes one node from that list:

`init → bash → sshd → apache → init`

becomes:

`init → bash → apache → init`

The `sshd` process is still running, it still has memory and the scheduler still runs it, But the kernel no longer reports it, because the kernel itself can't **see** it through its own structure.

The process is not hidden from reality, it is hidden from the kernel's representation of reality.

## Windows: `EPROCESS`

Windows uses a similar model:

```C
typedef struct _EPROCESS {
    LIST_ENTTRY ActuveProcessLinks;
    HANDLE UniqueProcessId;
    PVOID Token;
    ...
} EPROCESS;
```

Again, the key is:

```C
LIST_ENTRY ActiveProcessLinks;
```

Which is the Windows equivalent of Linux's `tasks`, literally the same idea, the same weakness, the same rootkit strategy.

## Memory Representation (Where ALl of This Lives)

None of these structures exist 'in files' or 'in binaries', they live in kernel memory, which is mapped seperately from user memory and it is protected by hardware (MMU + CPU), and also cannot be directly accesses by user programs.

When a rootkit manipulates kernel structures, it is directly modifying kernel memory, this is why:

* A wrong pointer = kernel panic
* A bad write = system crash
* A race condition = unreproducible instability

Kernel memory is not forgiving, this is why kernel rootkits are hard, and why most malware stays in userland.

## File Representation (What a File actually is)

A file is a collection of interconnected structures, linux uses a **Virtual File System (VFS)** to abstract all filesystems.

No matter if the file is on ext4, xfs, ntfs, or even tmpfs, the kernel always sees it through VFS structures.

## Inode (The File's Identity)

The inode basically contains metadata and behaviour.

```C
struct inode {
    umode_t i_mode;
    uid_t i_uid;
    gid_t i_gid;
    loff_t i_size;
    const struct file_operations *i_fop;
    ...
};
```

The inode knows permissions, sizes, and what function handle reads/writes.

Important thing is, there is no filename in the inode, the inode does not know what it is called, it only knows what it is.

## Dentry (The File's Name and Place)

The directory entry connects names to inodes;

```C
struct dentry {
    struct inode *d_inode;
    struct dentry *d_parent;
    struct list_head d_subdirs;
    ...
};
```

This is how `/etc/passwd` exists:

`"passwd" → inode #12345 `

A directory is just a list of dentries; To hide a file, a rootkit does not destroy the inode, it removes the dentry from the parent's list, like the fill still exists on the disk, but no path points to it anymore.

Again, it is not about hiding the reality, it is about breaking the representation.

## File Structure (An Open Instance)

When a process opens a file, the kernel creates a `file` structure:

```C
struct file {
    struct inode *f_inode;
    const struct file_operations *f_op;
    loff_t f_pos;
    ...
};
```

This represents:

*This process has this file open at this position, this is also where rootkits hook file behavior.*

## File Operations (Where Rootkits Intercept Files)

Look at this:
```C
struct file_operations {
    ssize_t (*read)(...);
    ssize_t (*write)(...);
    int (*open)(...);
    int (*release)(...);
};
```

There are function pointers, when you do `cat file.txt`, the kernel literally executes:

`file -> f_open -> read(...)`

*To intercept file reads, you can replace `read` with you function.*

It is not patching codes, not even changing binaries, it is just redirecting pointers, this is why rootkits love file_operations.

## Network Representation (What a "Connection" is)

A network connection is not a line in `netstat`, it is a structure in memory.

Linux uses sockets:

```C
struct socket {
    struct sock *sk;
    const struct proto_ops *ops;
    ...
};
```

And protocol operations:

```C
struct proto_ops {
    int (*connect)(...);
    int (*sendmsg)(...);
    int (*recvmsg)(...);
};
```

Once again, function pointers, **Rootkits** hide connections by:

* Filtering socket lists
* Intercepting protocol operations
* Manipulating network-related structures

The kernel never "knows" it is lying, it just executes altered logic.

## Credentials (What "Root" Actually Means)

There is no magic 'root mode', Root is just a set of fields inside a structure.

In Linux:

```C
struct cred {
    uid_t uid;
    uid_t euid;
    kernel_cap_t cap_effective;
    ...
};
```

When a program is root, it simply means:

```
cred -> uid == 0
cred -> euid == 0
```

Privilege Escalation at kernel level is literally:

* Duplicating a cred structure
* Changing those values
* Replacing the process's cred pointer

There is no authentication, no password or permission checks.
The kernel assumes its own structures are trusted.

Rootkits exploit that trust.

## Linked Lists (The Skeleton of the Kernel)

Almost everything in the kernel is linked using:

```C
struct list_head {
    struct list_head *next;
    struct list_head *prev;
};
```

Processes, Files, Drivers, Modules, Sockets, Everything.
The kernel is a massive linked structure graph.

Rootkits rarely destory objects, they unlink them, because unlinking them is:

* Stealthier 
* Reversible
* Non-destructive
* Less likely to crash the system

This is why DKOM works.

## Why This Matters for Rootkits

Once you understand kernel internals like:

* Process hiding is unlinking `task_struct`
* File hiding is removing dentries
* Privilege Escalation is modifying `cred`
* Syscall hooking is redirecting dispatch pointers
* VFS hooking is latering file_operations
* Network hiding is filtering socket structures

Rootkits ain't a fkn magic, they're just precise manipulation of trusted kernel data structures.

The kernel is not some code or anything, it is just data + rules, they modify the data, the rules operate on

And that is why:

* Detection is hard
* Stability is fragile
* Kernel exploitation is elite-tier

