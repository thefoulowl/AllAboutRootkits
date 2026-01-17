User Space Vs Kernel Space (why rootkits even work)

Modern CPUs implement privilege seperation. On x86 this is done using 
rings.

* Ring 3 (user space): normal programs 
* Ring 0 (kernel space): total control

User programs are not allowed to:

* touch hardware directly
* modify memory outside their processes
* schedule tasks
* change credentials

So how do user programs do anything useful? Right?

They ask the kernel to do it for 'em.

That request boundary is called a system call, it will later be 
discussed cleanly.

When a program asks "open this file", the kernel answers by:

1. Looking at internal kernel data
2. Performing operations
3. Returning a result

A rootkit lives where it can alter that answer.
