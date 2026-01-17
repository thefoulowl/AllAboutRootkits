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

There are only there fundamental ways to hook anything, and the 
process is known as hooking.

1. You overwrite instructions
2. You replace a pointer
3. You intercept a dispatch mechanism

Every hooking technique you will ever see reduces to one of these.

Inline hooks: rewriting reality
