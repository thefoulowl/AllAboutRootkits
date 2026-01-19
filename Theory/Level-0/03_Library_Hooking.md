# User-Space Function Hooking (Library/API Hooking)

## Core Idea

Instead of modifying the kernel, the attacker lies to programs by 
intercepting library functions they call.

### Program don't talk to the kernel directly, they usually go through:

* libc
* Win32 APIs (Windows)

If you hook those, you control what the program sees.

It's like, whenever a program answers a question, the rootkit answers 
instead of the OS, cool RIGHT!

For an example, we run 'ps aux' in our bash terminal in linux

Now, because of the rootkit interception, we get the answer minus the 
rootkit's process.

`[List of processes - rootkit process]`

### Where hooks live

* Shared libraries
* In-process memory
* Loader mechanisms

### Why is it basic

* No kernel access needed
* Easier to implement
* Easier to detect

### Defender View

* Compare program output vs raw syscalls
* Look for unexpected library loads
* Static + runtime integrity checks
