---
layout: default
title: getchar的背后
nav_order: 2
parent: Linux内核
author: qingan
---

{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>
# getchar -> syscall

## 1. getchar函数 -> read(@libc)调用路径
```C
$ less glibc-2.31/libio/getchar.c

int
getchar (void)
{
  int result;
  if (!_IO_need_lock (stdin))
    return **_IO_getc_unlocked** (stdin);
  _IO_acquire_lock (stdin);
  result = _IO_getc_unlocked (stdin);
  _IO_release_lock (stdin);
  return result;
}


-> alias _IO_getc_unlocked __getc_unlocked_body -> __uflow
@glibc-2.31/libio/bits/types/struct_FILE.h

-> _IO_UFLOW(fp) @ /glibc-2.31/libio/genops.c

-> _IO_JUMPS_FUNC(fp)->__uflow) (fp) @ glibc-2.31/libio/libioP.h

-> __GI__IO_default_uflow (fp=0x7fb2cce42a40 <_IO_2_1_stdin_>) @ genops.c:362

-> _IO_new_file_underflow (fp=0x7fb2cce42a40 <_IO_2_1_stdin_>) at fileops.c:517
517	  count = _IO_SYSREAD (fp, fp->_IO_buf_base,
518			       fp->_IO_buf_end - fp->_IO_buf_base);
   _IO_SYSREAD == fp->__read  （间接调用） -> __GI___libc_read 

-> __GI___libc_read (fd=0, buf=0x5555565206b0, nbytes=1024) at /sysdeps/unix/sysv/linux/read.c:26
```

## 2. read@libc -> syscall 调用路径
```C
  __libc_read @ read.c

     23 ssize_t
     24 __libc_read (int fd, void *buf, size_t nbytes)
     25 {
     26   return SYSCALL_CANCEL (read, fd, buf, nbytes);
     27 }
```

__libc_read对应的反汇编代码：

```C
0x00007f51d7204c7e in __GI___libc_read (fd=0, buf=0x555555e262a0, nbytes=1024) at ../sysdeps/unix/sysv/linux/read.c:26
26	  return SYSCALL_CANCEL (read, fd, buf, nbytes);

```

==>

```C
#define SYSCALL_CANCEL(...) \
  ({                                                                         \
    long int sc_ret;                                                         \
    if (SINGLE_THREAD_P)                                                     \
      sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__);                            \
    else                                                                     \
      {                                                                      \
        int sc_cancel_oldtype = LIBC_CANCEL_ASYNC ();                        \
        sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__);                          \
        LIBC_CANCEL_RESET (sc_cancel_oldtype);                               \
      }                                                                      \
    sc_ret;                                                                  \
  })
```

==> -> inline assembly code template, like:

```C

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
==> final code:

```C
(gdb) disass
Dump of assembler code for function __GI___libc_read:
   0x00007f51d7204c70 <+0>:	mov    %fs:0x18,%eax
   0x00007f51d7204c78 <+8>:	test   %eax,%eax
   0x00007f51d7204c7a <+10>:	jne    0x7f51d7204c90 <__GI___libc_read+32>
   0x00007f51d7204c7c <+12>:	syscall 
=> 0x00007f51d7204c7e <+14>:	cmp    $0xfffffffffffff000,%rax
   0x00007f51d7204c84 <+20>:	ja     0x7f51d7204ce0 <__GI___libc_read+112>
   0x00007f51d7204c86 <+22>:	repz retq 
   0x00007f51d7204c88 <+24>:	nopl   0x0(%rax,%rax,1)
   0x00007f51d7204c90 <+32>:	push   %r12
   0x00007f51d7204c92 <+34>:	push   %rbp

```

注意：上图中，syscall的number，来自%eax = 0 （%fs:0x18 -> %eax）。对这样syscall number的跟踪，从程序分析的角度很难；但是可以从convention的角度解决。

