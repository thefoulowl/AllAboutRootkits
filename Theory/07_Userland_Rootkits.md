# Userland rootkits: controlled deception

In userland, programs rely heavily on shared libraries like libc.
When a program calls `open()`, is it not invoking the kernel directly. 

It is calling: 

```
libc -> syscall wrapper -> kernel
```

If you insert your own library before libc, your function runs first. 
That is `LD_PRELOAD`.


### Your function may:

* hide filenames
* filter directory listings
* fake process lists
* deny access selectively

Userland rootkits do not hide from the system, they hide from the 
observer.

## The "Observer" vs. The "System"

When we say a userland rootkit hides from the observer, we mean it 
intercepts the tools and APIs that a human (or an automated 
monitoring tool) uses to see what’s happening. It doesn't actually 
remove its presence from the operating system's internal management; 
it just puts on a mask when someone asks to look at it. 

1. How it works: API Hooking:

Userland rootkits primarily operate through API Hooking (e.g., 
Inline hooking or IAT hooking).

* The Reality: The malicious process is running. It has a Process ID 
(PID), it consumes memory, and it has open handles.

* The Mask: When you open Task Manager or run ps in Linux, those 
programs call system APIs (like EnumProcesses in Windows or reading 
/proc in Linux). The rootkit intercepts that specific call and says, 
"Give them the list of processes, but delete my name from the list 
before they see it."

2. The Vulnerability: The "System" still knows

Because the rootkit lives in Ring 3 (User Mode), it is subject to 
the same rules as the tools trying to find it.

* The Kernel (Ring 0) still knows the process exists.

* The CPU is still executing its instructions.

* The scheduler is still giving it time slices.

If you ask the "System" directly (by using a kernel-mode driver or 
looking at raw memory/disk structures), the rootkit is plain as day. 
It is only hiding from the observer who relies on the hooked APIs.

### Comparison: Userland vs Kernel Rootkits

```
---------------------------------------------------------------------------------------------------------------------------------------------
     Feature	                     Userland Rootkit (Ring 3)	                             Kernel Rootkit (Ring 0)
---------------------------------------------------------------------------------------------------------------------------------------------

 Hiding Strategy       Intercepts library calls (DLL/Shared Object hooking).	Modifies internal kernel structures (DKOM).
 Visibility	           Hidden from top, ps, ls. Visible to kernel debuggers.	Hidden from the OS itself. The OS "forgets" it exists.
 Stability	           High. If it crashes, only the hooked app dies.      	        Low. If it crashes, you get a BSOD/Kernel Panic.
 Complexity            Relatively simple (LD_PRELOAD, IAT hooking).	                Highly complex (Requires driver signing bypasses).
---------------------------------------------------------------------------------------------------------------------------------------------
```

