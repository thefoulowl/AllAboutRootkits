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

*Why this hierarchy? It saves RAM; If a process only uses 1MB of memory, the kernel only needs to create a few entries in the tables, rather than a massive flat map for the entire 64-bit address space*

We'll discuss it further in our notes later on.
