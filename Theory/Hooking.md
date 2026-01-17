# What "hooking" really means

When getting thrown yourself into the rootkit world, you would've 
heard about hooks so much, we are going to discuss what are they and 
why do they even exists.

Hook is basically is a control-flow redirection.

### Every program, at every levelis just:

* Instructions
* Addresses
* Jumps
* Pointers

A hook changes where execution goes.

### There are only there fundamental ways to hook anything.

1. You overwrite instructions
2. You replace a pointer
3. You intercept a dispatch mechanism

Every hooking technique you will ever see reduces to one of these.

## Inline hooks: rewriting reality

When a function is called, execution jumps to its first instruction.

### An inline hook overwrites those first instructions with a jump:

```
original_function:
    push rbp
    mov rbp, rsp
    ...
```

#### Becomes 

```
original_function:
    jmp my_code
```

Now your code runs first and you may execute your logic, optionally 
then jump back to the original code.

This works because the CPU does not care who wrote the instructions.

### Inline hooks are powerful and dangerous:

* You must preserve registers
* You must respect calling conventions
* You must handle instruction length correctly

Kernel inline hooks are extremely fragile, one mistake equal panic.

## Pointer hooks: lying politely 
