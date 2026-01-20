# Virtual Memory - Where Kernel Reality Lives

## Core Idea

There is no such thing as "memory at address 0x12345678", that number does not point to RAM, it points to a translation mechanism.

Every address you see in code, in user programs or kernel code, it is all just a virtual address, not a physical one.

#### Virtual memory is the system that makes:

* Isolation possible
* Multitasking safe
* Kernel protections enforceable
* Rootkits either feasible or impossible

Without understanding Virtual Memory, kernel structures are unreachable ghosts.

## Why Virtual Memory Exists

Early computers used physical memory directly, every program accesses raw RAM and that led to:

* programs overwriting each other
* no isolation
* no security
* constant crashes

Modern systems solve this by *Never letting programs touch physical memory directly*. Instead, every process sees its own virtual address space, and the CPU translates it behind the scenes.

This translation is enforced by hardware, not software and this is one of the most important thing to understand

**The kernel does not "protect memory", the CPU does.**

## Virtual Memory vs Physical Memory

### Physical Memory (RAM)

Physical memory is a flat, linear array of bytes. Each byte has a hardwired address. However, modern software never sees this.

* **Fixed Blocks (Frames):** The Kernel and CPU treat physical RAM as a collection of fixed-size blocks called **Page Frames** (usually 4KB).

* **The MMU (Memory Management Unit):** This is the piece of hardware inside your CPU that does heavy lifting. Every time your code tries to access an address, the MMU intercepts it. If the mapping doesn't exist in the **Page Tables**, the CPU triggers a `Page Fault` (Interrupt 14 on x86), handling control back to the kernel to "fix" it (or kill the process).

#### The Translation Mechanism: Page Tables

Think of Page Tables as a multi-level phone book. On x86_64, this is usually a 4-level or 5-level hierarchy. A virtual address is actually a set of indexes into these tables:

1. **Level 4 (PML4):** Page Map Level 4 (The top-level index).
2. **Level 3 (PDPT):** Page Directory Pointer Table
3. **Level 2 (PD):** Page Directory
4. **Level 1 (PT):** Page Table
5. **Offset :** The specific byte within the 4KB page.

*Why this hierarchy? It saves RAM; If a process only uses 1MB of memory, the kernel only needs to create a few entries in the tables, rather than a massive flat map for the entire 64-bit address space*. We'll discuss it further in our notes later on.

```
Virtual Address
   ↓
PML4
   ↓
PDPT
   ↓
PD
   ↓
PT
   ↓
Physical Page
```

Each level is a table that:

* points to the next
* or denies access

This is what makes:

* kernel memory inaccessible to user processes
* user memory invisible to other processes
* memory permissions enforceable

With this translation machinery in place, the OS can now construct the illusion known as Virtual Memory.

### Virtual Memory 

Virtual Memory is a memory management technique that provides an 'idealized' view of storage to a process. Every process on the system thinks it has a massive, contiguous, and private range of addresses, regardless of how much physical RAM is actually installed.

```
The system never maps bytes to bytes.
It maps:
Virtual Pages → Physical Frames
```

**1. Isolation (The Sandbox):**

In a modern OS, **Process A cannot see Process B's memory.** Both processes might think they are storing data at address `0x400000`, but the MMU maps them to completely different physical page frames. 

* **Security Perspective:** This is the first line of defense. Without virtual memory, any program could read the memory of your password manager or the kernel itself just by guessing the right physical address.

**2. The Address Space:**

On a 64-bit system, the theoretical virtual address space is 2^64 bytes
(though most CPUs currently limits to 48 or 57 bits).

* **User Space:** Where your application code, heap, and stack live.
* **Kernel Space:** The upper portion of the address space reserved for the OS.
* **The 'Gap':** Most of this space is actually "unmapped", if a process tries to access an unmapped address, the MMU sees no entry in the Page Tables and triggers a **Page Fault** which then kernel may turn into a **Segfault**.

**3. Over-Commitment and Swapping:** 

Virtual memory allows the system to expose more addressable memory than available physical RAM, using disk as a backing store and lazy allocation.

* **Demand Paging:** The kernel only maps a virtual page to a physical frame when the process actually tries to use it.

* **Swapping/Paging Out:** If physical RAM is full, the kernel can move a page frame to the disk (Swap space) and mark the Page Table entry as "not present". When the process tries to access it again, a **Page Fault** occurs, and the kernel quietly swaps it back into RAM.

## Why Userland Cannot Access Kernel Memory

Kernel memory pages are marked as:

`Supervisor-only`

Which means:

* Ring 3 (user) code -> forbidden
* Ring 0 (kernel) code -> allowed

If userland tries:

`*(int*)0xffff888000000000 = 123;`

The CPU detects access to supervisor page and raises a fault and then the kernel kills the process, no syscall, no warning, just termination. And this is why kernel rootkits must run in kernel space, because only Ring 0 code can legally touch kernel memory.

## Kernel Memory Layout (Conceptual)

Simplified view on x86_64 Linux:

```
0x0000000000000000 ─────────► User space start
...
0x00007fffffffffff ─────────► User space end
--------------------------------------------
0xffff800000000000 ─────────► Kernel space start
...
0xffffffffffffffff ─────────► Kernel space end
```

User and Kernel share the same address space layout, but permissions differ. So, same address space with different access rights and this is why kernel code can access user memory but user code cannot access kernel memory.

## How Kernel Structures Live In Memory

Everything from `task_struct` to `inode` lives in kernel virtual memory.

When a rootkit:

* Unlinks a process
* Modifies credentials
* Hooks file operations

It is directly modifying kernel memory pages and that is why even a very little mistake can lead to kernel panic, kernel memory is **live** not static.

## Memory Protection: That Shape Rootkit Design

Modern systems use several hardware protections that deeply affect rootkits:

**NX (Non-Executable):**

It is like, a mark when assigned, makes that specific part of the memory non-executable.

So, data pages cannot run code and code injection into data memory fails. Rootkits must place the code in executable memory or just disable NX which is quite risky.

**SMEP (Supervisor Mode Execution Prevention):**

It prevents the Kernel from executing code located in user memory, so this blocks jumping to userland shellcode from kernel and trivial kernel exploitation. Rootkits must ensure all code lives in kernel memory and not in user pages.

**SMAP (Supervisor Mode Access Prevention):**

It prevents the Kernel from accessing user memory unless explicitly allowed, and this blocks Kernel reading user buffers freely and many exploitation techniques. Rootkits must carefully enable/disable SMAP or avoid touching user memory directly.

**KASLR (Kernel Address Space Layout Randomization):**

It randomizes the kernel memory layout on boot, this means addresses of structures change every reboot and static offsets no longer work, it is the same as ASLR but for the kernel memory. Rootkits must dynamically locate kernel symbols or infer the structures at runtime. *Memory randomness is a direct defense against rootkits.*

## Why Memory is the True Battlefield:

Files can be deleted, processes can be restarted, and disks can be wiped, but while the system is runnning, memory is the only place where reality exists. Rootkits that never touch disk, live only in the memory, and hook live structures are harder to detect, harder for forensically recover and even harder to prove, that is why advanced rootkits aim to be memory-only.

## Why we cannot **Scan Memory** Easily

Memory is fragile, fragmented, constantly changing and even context-sensitive, and a structure may move, be freed, be reallocated and may change size too, this is why naive memory scanning fails and tools like Volatility uses signatures and the defenders struggle with memory-only implants. Memory ain't a storage it's a flow.

## Rootkits and Memory Allocation 

Kernel memory is allocated via:

* kmalloc
* vmalloc
* slab allocators

Rootkits must:

* allocate memory in ways that look legitimate
* avoid suspicious patterns
* clean up properly

Bad memory handling leaves traces, causes crashes and even exposes implant and this is why many rootkits reuse existing kernel memory and avoid fresh allocations when possible.