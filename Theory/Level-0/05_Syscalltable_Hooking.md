# System Call Table Hooking (Kernel-Level)

## Core idea 

Every system call (read, open, kill, etc.) is dispatched via a table 
of function pointers in the kernel.

Rootkits modify that table, it would be like, all roads to the kernel 
would go through us.

## Where the Syscall Table Fits in the System

*Before understanding why syscall table hooking is powerful, we must 
understand **where it actually sits in the execution flow**.*

### When a user-space program makes a system call like:

````C
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

### Instead of:

```
sys_open → real kernel function
```

### It becomes:

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
