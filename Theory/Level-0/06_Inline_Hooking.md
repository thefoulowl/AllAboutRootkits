# Inline Hooking (Code Patching/Trampolines)

## Core Idea

Instead of changing tables, the rootkit rewrites the first instruction 
of a function. It is like, run my code first, then do whatever u want.

## Inline Hooking vs Table Hooking: Code vs Data Mutation

There is a fundamental architectural differece between inline hooking and the table-based hooking that matters deeply for detection, stability, and stealth.

Inline hooking modifies code itself.

Table-based hooking (syscall table, file_operations, etc.) modifies **data structures** that point to code and this distinction is critical.

**Inline hooking (Code Mutation)
When you inline hook, you are:

* Overwriting instructions in executable memory
* Changing the actual behaviour of a function at its entry point
* Mutating what the CPU executes directly

This means:

* Integrity checks on code can detect it
* Updates or recompilations break it easily
* You must preserve instruction boundaries perfectly

Inline hooking is invasive because it alters **what the CPU runs**.

## Table Hooking (Data Mutation)

When you hook via tables, you are:

* Leaving the original code untouched
* Redirecting which function pointer is used
* Altering the dispatch login rather than the execution logic

This means:

* The original code remains clean in memory
* You are manipulating kernel state, not kernel text
* Detection shifts from code integrity to data integrity

Table hooking is subtle because it alters **what the OS chooses to run**, not how the code itself runs.

#### Why this difference matters for rootkits

Rootkits prefer data mutation over code mutation because:

* Code mutation breaks easily
* Code mutation is more detectable
* Code mutation is architecture-fragile

This is why Inline Hooking is powerful but DKOM and pointer hooking dominate serious rootkit design

Inline hooks are used no dispatch table exists or when the rootkit must intercept execution before any table is consulted

This distinction will later explain:

Why DKOM is stealthier than inline hooking
Why PatchGuard targets code modification aggressively
Why modern rootkits avoid patching kernel text
Why eBPF and function redirection are preferred today

### Original: 

```
function:
  push rbp
  mov rbp, rsp
  ...
```

### Hooked:

```
function:
  jmp rootkit_handler
```

### Why attackers like it

* No global table changes
* Works in user or kernel space
* Harder to notice at a glance

### Why is it fragile

* Instruction size matters
* CPU architecture matters
* Updates break it

### Defender View:

* Function prologue integrity checks
* Code section hashing
* Detect unexpected jumps
