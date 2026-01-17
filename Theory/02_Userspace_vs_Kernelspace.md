# User Space Vs Kernel Space (why rootkits even work)

Modern CPUs implement privilege seperation. On x86 this is done using 
rings.

* Ring 3 (user space): normal programs
* Ring 0 (kernel space): total control

### User programs are not allowed to:

* touch hardware directly
* modify memory outside their processes
* schedule tasks
* change credentials

So how do user programs do anything useful? Right?

They ask the kernel to do it for 'em.

### That request boundary is called a system call, it will later be 
### discussed cleanly. When a program asks "open this file", the kernel 
### answers by:

* Looking at internal kernel data
* Performing operations
* Returning a result

### A rootkit lives where it can alter that answer.

## 1. First principles: why spaces exist at all

### A CPU does not naturally know what "user" or "kernel" is, what is 
### does know is just this:

* Privilege levels (rings/modes)
* Memory permissions 
* Instruction permissions

### The OS uses these hardware features to build an illusion:

"This code is allowed to do everything, that code isn't."

That illusion is what we call user space vs kernel space.

## 2. Kernel space: what it REALLY is

Kernel space is basically a privilege contract enforced by hardware.

### Kernel space means:

* CPU is in privileged mode
* Code can :
	* Execute privileged instructions
	* Access all physical memory
	* Program MMU, CPU tables, devices
	* Control scheduling, interrupts, I/O

### What actually runs in kernel space:

* The kernel itself
* Loadable Kernel Modules (drivers, LKM rootkits)
* Interrupt handlers
* Syscall handlers
* Scheduler 
* VFS, network stack, memory manager

### If kernel code screws up, it won't lead to SEGFAULT, it will lead 
### to SYSTEM crash or KERNEL PANIC!
### This alone tells you how much power it has.

## 3. User space: what it REALLY is

User space is deliberately crippled execution.

### User space means:

* CPU in unprivileged mode
* Code:
	* Cannot access hardware directly
	* Cannot touch kernel memory
	* Cannot execute privileged instructions
	* Cannot disable protections

### What runs in user space:

* Shells
* Applications
* Services (nginx, sshd)
* Malware
* Exploits
* You

### User space cannot be trusted by design, even `ROOT` is still user 
### space, Root != Kernel, Root is just a privileged user-space 
### identity.

## 4. Memory seperation (Critical)

Let's talk virtual memory, because this where the real seperation 
happens.

### Every process gets:

* Its own virtual address space
* Same virtual addresses != same physical memory


### Typical Linux Layout in simplified format: 

``` 
0x00000000 ───────────────► user code
0x40000000 ───────────────► user heap / stack
0x7fffffffffff ──────────► end of user space
--------------------------------------------
0xffff800000000000 ──────► kernel space
```

### Key facts:

* User space cannot map kernel pages
* Kernel pages are:
	* Marked supervisor-only
	* Enforced by MMU + CPU

### Trying to access kernel memory from user space:

* Results in page fault
* Kernel kills the process

## 5. How user code talks to kernel code (Gateway)

There are only 3 legitimagte entry points from user space into kernel 
space:

### A. System calls:

Controlled and audited entry points.

#### Example:

```C
read(fd, buf, size)
```

#### Flow:

```
user code
 └─► libc wrapper
     └─► syscall instruction
         └─► kernel syscall handler
             └─► kernel function
```

Kernel validates the arguments, pointers, permissions.

### B. Interrupts:

Interrupts are hardware-generated and user space cannot fake hardware 
interrupts.

#### Examples:

* Keyboard Input
* Network packet

### C. Exceptions/faults:

* Page fault
* Divide-by-zero
* Invalid instruction

These force kernel entry.

## 6. Context switch: the invisible boundary crossing

### When entering kernel space:

* CPU switches mode
* Stack changes (user stack -> kernel stack)
* Registers are saved
* Execution jumps to kernel code

This is not a function call, this is a privilege transition enforced 
by silicon.

### Returning to user space:

* CPU restores user registers
* Switches privilege
* Continues execution

If this goes wrong then KERNEL PANIC!

## 7. Why Kernel Exploits are different typa monsters

### User-space exploits:

* You control your process
* You crash -> you die 

### Kernel exploit:

* You control the OS
* You crash -> everyone dies

### What kernel exploits aim to do:

* Escalate privileges
* Disable protections (SELinux, SMAP, SMEP)
* Hook syscalls
* Hide processes/files
* Persist invisibly

That's why kernel bugs are gold!

## 8. Rootkits vs Spaces

### User-space rootkits:

* Hook libc (LD_PRELOAD)
* Hook binaries 
* Hide from naive tools
* Easy to detect with statically linked tools

### Kernel-space rootkits:

* Hook syscalls
* Hook VFS
* Hook network stack
* Modify kernel data structures
* Can lie to every user-space tool

This is why serious stealth must go KERNEL!

## 10. Final model 

```
User space is sandboxed execution.
Kernel space is absolute authority.
The boundary is enforced by hardware, not software.
The syscall interface is the only door.
Rootkits exist to own the side that lies.
```

