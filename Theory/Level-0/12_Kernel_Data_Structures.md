# Kernel Data Structures (How the Kernel Organizes everything)

The kernel is not a collection of independent objects, it is a living web of interconnected structures, nothing in the kernel exists in isolation because:

* Processes are linked to other processes
* Files are linked to directories
* Sockets are linked to protocols
* Drivers are linked to devices
* Memory regions are linked to owners

This connectivity is enforced entirely through **data structures**. Rootkits do not attack code first, they attack the structures that decide how code is reached. If you understand kernel data structures, rootkits stop being mysterious, they're just graph manipulation.

## Why Data Structures Matter More Than Code

Code defines the behaviour of the program but the data strctures define visibility, reachability, and order. A function that is never reached does not matter, and a structure that is unlinked does not exist, as far as the kernel is concerned.

Rootkits exploit this by:

* Removing nodes from traversal paths
* Replacing pointers
* Redirecting structure relationships

This is more powerful than patching the code, because code can be checked but data changes are harder to validate globally.

## Linked Lists (The Skeleton of the Kernel)

The most fundametal kernel structure is the doubly-linked list, almost everything in the kernel is stored in lists, processes, modules, drivers, open files, sockets and even the memory regions.

### **The Basic Structure:**

```C
struct list_head {
    struct list_head *next;
    struct list_head *prev;
};
```
This looks simple, but it forms the backbone of kernel organization. A list of objects look like `A ←→ B ←→ C ←→ D`. Each object embeds a `list_head` inside itself.

The kernel never stores like an array or anything, it stores like, here is a note that links to another node, that links to another node and so on. Basically, traversal happens by following pointers.

## Circular Lists (Why the Kernel Uses Them)

Kernel lists are often are circular, instead of `A → B → C → NULL`, they look like `A → B → C → A`. This means that there is no NULL terminator and every node has both prev and next, the list is never 'empty', it just loops back.

### Why this matters?

This design avoids NULL pointer checks while simplifying the insertion and removal, and also ensures stable traversal even under mutation. Rootkits exploit this by removing a node without breaking the ring, so the traversal continues safely, just skipping the hidden object.

## How Process Hiding Works at Structure Level

The kernel stores processes as a circular linked list of `task_struct`
When tools walk processes, they literally do:

```C
for_each_process(task) {
    // print info
}
```

Which internally follows:

`task->tasks.next → next task_struct`

To hide a process, you don't kill or suspend it, you unlink its node, conceptually it is like `A ←→ B ←→ C ←→ D`, remove C, `A ←→ B ←→ D`

C still exists, it still runs but it is simply unreachable from the traversal path. This is invisibility at the kernel level.

## Hash Tables (How the Kernel Finds Things Fast)

Linked lists are good for traversal, but slow for lookup, so the kernel also uses hash tables, these are used for:

* PID lookup
* Inode lookup
* File Descriptor tables
* Socket lookup
* Credential caching

Instead of walking a list of 100,000 processes, the kernel:

* hashes the PID
* jumps directly to a bucket
* and finds the object in constant time

### Why this matters for rootkits

If you unlink a process from the task list but forget to:

* remove it from PID hash table
* remove it from scheduler queues

Then the kernel still knows about it internally and inconsistencies appear, Advanced rootkits must **modify all structure references**, not just one list. This is why naive DKOM causes crashes or detection.

## Trees (Ordering and Hierarchy)

Some kernel subsystems use trees instead of lists

### Red-Black Trees

The kernel uses Red-Black trees for:

* Virtual memory areas (VMAs)
* Process Scheduling
* File system caches

These trees maintain ordering, balance, fast lookup. A memory region is not just stored arbitrarily. It is placed in a tree ordered by virtual addresses.