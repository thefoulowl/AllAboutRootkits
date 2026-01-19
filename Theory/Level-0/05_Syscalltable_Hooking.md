# System Call Table Hooking (Kernel-Level)

## Core idea 

Every system call (read, open, kill, etc.) is dispatched via a table of function pointers in the kernel.
Rootkits modify that table, it would be like, all user-initiated kernel services go through this dispatch layer.

## Where the Syscall Table Fits in the System

*Before understanding why syscall table hooking is powerful, we must 
understand **where it actually sits in the execution flow**.*

### When a user-space program makes a system call like:

```C
read(fd, buf, size);
```

It doesn't jump directly to a kernel function like `sys_read()`, 
instead, the flow is:

```
User Program
   ↓
libc wrapper (read())
   ↓
syscall instruction (CPU trap)
   ↓
Kernel syscall entry point
   ↓
Syscall dispatcher
   ↓
syscall_table[__NR_read]
   ↓
sys_read() (or equivalent handler)
```

## What the syscall table really is

The syscall table is simply an **array of function pointers,** indexed by syscall number.

### Conceptually: 

```
syscall_table[__NR_read] = sys_read;
syscall_table[__NR_open] = sys_open;
syscall_table[__NR_kill] = sys_kill;
```

### So when a program executes a syscall:

* The kernel looks up the syscall number
* Uses it as an index
* Calls the function pointer stored there

The syscall table is basically just a dispatch table.

## Why SysCall Hooking is so powerful

Because every user to kernel or like every user's kernel operation must pass through this dispatch layer, hooking here means:

* You don't lie to tools
* You lie to the kernel itself
* You change the OS's interpretation of reality

*A hooked syscall table does not:*

* Hide from `ps`
* Hide from `ls`
* Hide from `netstat`

It hides from the kernel's own internal logic and that is why syscall hooking is qualitatively different from:

* LD_PRELOAD
* Binary kit
* API hooking

Those lie to the observers and syscall hooking lies to the system.

## Why modern Kernels protect it

*Because the syscall table is like the central to kernel integrity, used by every process and it is also required for system stability*

#### Modern kernels:

* Mark it read-only
* Randomize its location (KASLR)
* Detect modification attempts

So, syscall hooking is not just 'Advanced' because it is kernel-level, it is advanced because the OS actively defends it.

You should think of syscall table hooking as - ***Hooking at the dispatch layer rather than the execution layer***

Execution layer = inline hooking
Dispatch layer = syscall table hooking

#### Instead of:

```
sys_open → real kernel function
```

#### It becomes:

```
sys_open → rootkit function → (optional) real kernel function
```

### What this allows

* Hide processes
* Hide network connections
* Hide files
* Backdoor privilege checks

### Why is it more advanced

* Requires kernel access
* Kernel crashes if done wrong
* Modern kernels protect this heavily

### Defender View

* Syscall table integrity validation
* Kernel memory write-protection checks
* Behaviour mismatches between syscall vs hardware traces
