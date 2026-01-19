# Inline Hooking (Code Patching/Trampolines)

## Core Idea

Instead of changing tables, the rootkit rewrites the first instruction 
of a function.

It is like, run my code first, then do whatever u want.

### Original: 

```
function:
  push rbp
  mov rbp, rsp
  ...
```

### Hooked:

```
function:
  jmp rootkit_handler
```

### Why attackers like it

* No global table changes
* Works in user or kernel space
* Harder to notice at a glance

### Why is it fragile

* Instruction size matters
* CPU architecture matters
* Updates break it

### Defender View:

* Function prologue integrity checks
* Code section hashing
* Detect unexpected jumps
