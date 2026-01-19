# Introduction 

A modern operating system is not a single program, it is a layered 
contract system.

At the bottom is hardware. The CPU does not know what a "process" or 
"file" is. It only knows instructions, memory, registers, and 
privilege rings.

The operating system kernel exists to lie politely to user programs, 
it pretends the machine is safer, more structured, and more isolated 
than it really is.

Everything you see, files, directories, processes, sockets, 
permissions, it is a representation, not reality.

### Rootkits exploit the gap between:

* Reality (what actually exists)
* Representation (what the OS reports)

### If you remember only one sentence, remember this that -

A rootkit does not hide things.
It controls the answers to the questions.


