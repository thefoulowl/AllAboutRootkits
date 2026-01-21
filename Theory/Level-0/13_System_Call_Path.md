# System Call Path (How User Code Enters Kernel Reality)

## The Boundary That Matters

There is exactly one legitimate way for user programs to request kernel services. Not `open()`, `read()`, `kill()`, the real boundary is a CPU-enforced privilege transition. A system call is not a function call, it is a controlled violation of privilege, approved and supervised by hardware. 

If we don't understand this path, syscall hooking would look like a trick, and once we do, it becomes so obvious why this works and where it must be intercepted.

## The Fundamental Rule

User space cannot execute privileged instructions or access kernel memory or change any hardware states. Yet user programs still can read files, create processes, send packets and even kill processes, they do it by asking, not by doing and the syscall path is the asking mechanism.

## The Illusion of a Function Call

When you write `read (fd, buf, size);`, it looks like a normal function call but it is not, this line does not jump directly into the kernel. Instead, it starts a multi-stage transition involving:

* User libraries
* CPU registers
* Special instructions
* Kernel entry code
* Dispatch tables
* Internal subsystems

Let's walk it end to end.

## Stage 1: User Program -> libc Wrapper

Your program does not know how to talk to the kernel, so it calles a **library wrapper**, usually in libc.

For `read()`:

* the wrapper prepares the arguments
* places them into registers
* loads the syscall number

Nothing privileged happens here, this is still **Ring 3** code.

## Stage 2: Register Preparation (Critical Step)

On x86_64 Linux, syscalls registers like this:

```ini
RAX = syscall number (__NR_read)
RDI = arg1 (fd)
RSI = arg2 (buf)
RDX = arg3 (size)
R10 = arg4
R8  = arg5
R9  = arg6
```

This convention matters because:

* the kernel expects arguments only here
* nothing else is trusted
* stack contents are irrelevant at this boundary

The kernel does **not** trust user memory, user stack, or user state.

Only the registers.

## Stage 3: The `syscall` Instruction (The Point of No Return)

The libc wrapper executes `syscall`. Now this is not a jump, this is a CPU trap.

What exactly happens instantly:

1. CPU switches from Ring 3 -> Ring 0
2. CPU switches stacks (user stack -> kernel stack)
3. CPU saves user state
4. CPU jumps to a kernel-defined entry point

This transition is enforced by silicon, user code cannot choose where it lands or skip the validation or alter any typa privilege rules. At this moment, user space ceases to exist.

## Stage 4: Kernel Entry Stub

Execution now begins in a **kernel entry routine**, this code saves the registers, sanitizes state, disables or manages any interrupts and prepares a safe kernel context. Also, this is where the SMEP, SMAP, stack switching and context setup are enforced.

No syscall logic happens yet, this is pure boundary handling.

## Stage 5: Syscall Dispatcher

Now the kernel must answer one question, **Which service was requested?** and the answer is in `RAX` register. The kernel uses this value to index into the **syscall dispatch mechanism**.

Conceptually:

`handler = syscall_table[RAX]`

This is the **exact moment syscall table hooking targets**. Nothing filesystem-related has happened yet and nothing process-related happened yet. This is pure dispatch.

## Stage 6: Syscall Table Lookup

The syscall table is simply an array of function pointers, each entry maps: `syscall number → kernel handler`. For example:

```
__NR_read → sys_read
__NR_open → sys_open
__NR_kill → sys_kill
```

The kernel does **not** check:

* "Is the pointer safe?"
* "Has this been modified?"

It assumes that kernel memory is trusted, and this assumption is what rootkits exploit.

## Stage 7: Syscall Handler Execution

Now the actual syscall handler runs, for `read()`:

```
sys_read()
```

But this function still does not touch disk, it validates the file descriptor, resolves the file object and hands control to the VFS layer. The syscall handler is a **router**, not a worker.

## Stage 8: Subsystem Dispatch (VFS, Net, Proc, etc.)

From here, execution moves into subsystems:

* File syscalls -> VFS -> filesystem -> driver
* Network syscalls -> socket layer -> protocol -> driver
* Process syscalls -> scheduler -> task structures

Each layer follows pointers, walks structures, and calls function tables. This is where `VFS hooking`, `file_operations hooking`, `proto_ops hooking` come into play.

Syscall hooking happens before this, subsystem hooking happens after this.

## Stage 9: Return Path (Kernel -> User)

Once the operation is complete:

1. Return value is placed in `RAX`
2. Kernel restores saved state
3. CPU switches back to Ring 3
4. Execution resumes after `syscall`

From user space, it looks like:

```
read() returned 42;
```

From the system's perspective, a full privilege transition occured.

## Why Syscalls Are the Perfect Hook Point

Syscalls are mandatory, centralized, hardware-enforced, trusted by the design. Every serious kernel action initiated by user space must cross this path. This make syscall interception powerful, dangerous and heavily protected which is why modern kernels defend it aggressively.

## Why Not Everything Goes Through Syscalls

Not all kernel activity uses syscalls, the following do **not** go through the syscall table:

* Interrupts
* Kernel Threads
* Timers
* Drivers
* Internal Kernel Logic

This is why syscall hooking alone is insufficient, DKOM exists and kernel object manipulation matters. Syscalls are entry points, not total control.

## How This Connects to Rootkit Design

* **Hooking** -> redirecting execution
* **Syscall table hooking** -> intercepting dispatch
* **Inline hooking** -> intercepting execution
* **Kernel internals** -> knowing what handlers manipulate
* **Virtual memory** -> knowing where this all lives
* **Data structures** -> knowing what gets traversed

The syscall path is the spine that connects them.

## Why This Boundary Is So Heavily Defended

Modern defenses focus here because:

* corurption here affects everything
* stability risk is massive
* trust assumptions collapse

This is why:

* syscall tables are read-only
* addresses are randomized
* integrity checks exist
* PatchGuard exists (Windows)

Defenders guard the boundary because if it falls, everything above it lies.

