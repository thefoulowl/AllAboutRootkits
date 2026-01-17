# Userland rootkits: controlled deception

In userland, programs rely heavily on shared libraries like libc.
When a program calls `open()`, is it not invoking the kernel directly. 

It is calling: 

```
libc -> syscall wrapper -> kernel
```

If you insert your own library before libc, your function runs first. 
That is `LD_PRELOAD`.


