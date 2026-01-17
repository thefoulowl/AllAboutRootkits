What "hooking" really means

When getting thrown yourself into the rootkit world, you would've 
heard about hooks so much, we are going to discuss what are they and 
why do they even exists.

Hook is basically is a control-flow redirection.

Every program, at every levelis just:

* Instructions
* Addresses
* Jumps
* Pointers

A hook changes where execution goes.

There are only there fundamental ways to hook anything, and this 
process is known as hooking.

The core idea about hooking is that instead of modifying the kernel, 
the attacker lies to programs by intercepting library functions they 
call.

Program don't talk to the kernel directly, they usually go through:

* libc
* Win32 APIs (Windows)

If you hook those, you control what the program sees.

It's like, whenever a program answers a question, the rootkit answers 
instead of the OS, cool RIGHT!

For an example, we run 'ps aux' in our bash terminal in linux

Now, because of the rootkit interception, we get the answer minus the 
rootkit's process.

`[List of processes - rootkit process]`





