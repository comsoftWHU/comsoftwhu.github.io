---
layout: default
title: malloc的背后
nav_order: 2
parent: Linux内核
author: qingan
---

# malloc -> syscall


```
$ less glibc-2.31/malloc/malloc.c

__libc_malloc:
    ...
    if (SINGLE_THREAD_P)
    {
      victim = _int_malloc (&main_arena, bytes);
      assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
              &main_arena == arena_for_chunk (mem2chunk (victim)));
      return victim;
    }

    arena_get (ar_ptr, bytes);

    victim = _int_malloc (ar_ptr, bytes);
    /* Retry with another arena only if we were able to find a usable arena
        before.  */
    ...

-> __GI___libc_malloc (bytes=10) at malloc.c:3031

-> _int_malloc (av=av@entry=0x7f2efffcace0 <main_arena>, bytes=bytes@entry=1048576) at malloc.c:4141

-> sysmalloc (nb=nb@entry=1048592, av=av@entry=0x7f2efffcace0 <main_arena>) at malloc.c:2310

-> __GI___mmap64 (addr=addr@entry=0x0, len=len@entry=1052672, prot=prot@entry=3, flags=flags@entry=34, fd=fd@entry=-1,
    offset=offset@entry=0) at ../sysdeps/unix/sysv/linux/mmap64.c:59

-> return (void *) MMAP_CALL (mmap, addr, len, prot, flags, fd, offset); @mmap64.c:59

-> inline assembly code template like:

#define internal_syscall1(number, err, arg1)                            \
({                                                                      \
    unsigned long int resultvar;                                        \
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);                              \
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;                   \
    asm volatile (                                                      \
    "syscall\n\t"                                                       \
    : "=a" (resultvar)                                                  \
    : "0" (number), "r" (_a1)                                           \
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);                        \
    (long int) resultvar;                                               \
})

```

-> the final code is like: 

```
   │0x7faf0d015708 <__GI___mmap64+40>       je     0x7faf0d015760 <__GI___mmap64+128>      |
   │0x7faf0d01570a <__GI___mmap64+42>       mov    %rbx,%r9                                |
   │0x7faf0d01570d <__GI___mmap64+45>       mov    %r15d,%r8d                              |
   │0x7faf0d015710 <__GI___mmap64+48>       mov    %r14d,%r10d                             |
   │0x7faf0d015713 <__GI___mmap64+51>       mov    %r12d,%edx                              |
   │0x7faf0d015716 <__GI___mmap64+54>       mov    %r13,%rsi                               |
   │0x7faf0d015719 <__GI___mmap64+57>       mov    %rbp,%rdi                               |
   │0x7faf0d01571c <__GI___mmap64+60>       mov    $0x9,%eax                               |
   │0x7faf0d015721 <__GI___mmap64+65>       syscall                                        |
   │0x7faf0d015723 <__GI___mmap64+67>       cmp    $0xfffffffffffff000,%rax                |



```