---
layout: default
title: systemCall流程
nav_order: 1
parent: Linux内核
author: qingan
---

# system call流程
## 1 例子
```
$vim hello_syscall.c
#include <unistd.h>
#include <sys/syscall.h>
int main(void) {
        getchar();
        syscall(SYS_write, 1, "hello, world!\n", 14);
  return 0;
}
```

## 2 strace输出
```
$ strace ./hello_syscall.exe
**execve**("./hello_syscall.exe", ["./hello_syscall.exe"], 0x7ffd5cc823a0 /* 32 vars */) = 0
brk(NULL)                               = 0x556f56697000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
**openat**(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=73810, ...}) = 0
mmap(NULL, 73810, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f7dcc0d1000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
**openat**(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\260\34\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2030544, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f7dcc0cf000
mmap(NULL, 4131552, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f7dcbacc000
mprotect(0x7f7dcbcb3000, 2097152, PROT_NONE) = 0
mmap(0x7f7dcbeb3000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e7000) = 0x7f7dcbeb3000
mmap(0x7f7dcbeb9000, 15072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f7dcbeb9000
close(3)                                = 0
arch_prctl(ARCH_SET_FS, 0x7f7dcc0d04c0) = 0
mprotect(0x7f7dcbeb3000, 16384, PROT_READ) = 0
mprotect(0x556f54b0f000, 4096, PROT_READ) = 0
mprotect(0x7f7dcc0e4000, 4096, PROT_READ) = 0
munmap(0x7f7dcc0d1000, 73810)           = 0
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 1), ...}) = 0
brk(NULL)                               = 0x556f56697000
brk(0x556f566b8000)                     = 0x556f566b8000
**read**(0, "\n", 1024)                     = 1
**write**(1, "hello, world!\n", 14hello, world!
)         = 14
**exit_group**(0)                           = ?
+++ exited with 0 +++

```
## 3 分析
### 3.1 流程
1. shell调用execve，加载./hello_syscall.exe程序；
2. execve从exe文件的入口.interp段找到动态链接器ld.so的路径信息，并加载执行ld.so(如果内存中已经存在ld.so.cache，则从内存中打开)
3. ld.so从exe文件获得其依赖的libc.so库的路径信息，并加载libc.so
4. 初始化exe对应进程状态，并跳转到entry开始执行
5. getchar()代码调用syscall read
6. syscall调用syscall write
7. exit退出

### 3.2 system call入口分析

app -> libc ->syscall入口：

比如, hello_syscall -> syscall@libc -> write@syscall
```

1）hello_syscall -> syscall@libc
$ objdump -T ./hello_syscall.exe |grep syscall
./hello_syscall.exe:     file format elf64-x86-64
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 syscall // use


$ objdump -T libc.so|grep syscall
0000000000037d70 g    DF .text	0000000000000033  GLIBC_2.2.5 syscall // define

$ less /glibc-2.31//sysdeps/unix/sysv/linux/x86_64/syscall.S
.text
ENTRY (syscall)
        movq %rdi, %rax         /* Syscall number -> rax.  */
        movq %rsi, %rdi         /* shift arg1 - arg5.  */
        movq %rdx, %rsi
        movq %rcx, %rdx
        movq %r8, %r10
        movq %r9, %r8
        movq 8(%rsp),%r9        /* arg6 is on the stack.  */
        **syscall**                 /* Do the system call.  */
        cmpq $-4095, %rax       /* Check %rax for error.  */
        jae SYSCALL_ERROR_LABEL /* Jump to error handler if error.  */
        ret                     /* Return to caller.  */

PSEUDO_END (syscall)
```

2）hello_syscall -> **syscall@libc** -> **system_call(kernel)**
syscall指令跳转到内核入口函数system_call（地址由syscall_init()写入进MSR_LSTAR寄存器）：

函数从用户态切换内核态；将用户态传来的参数从寄存器传到内核栈；根据syscall number跳转到sys_call_table（根据arch/x86/syscalls/syscall_64.tbl建立跳转表）中对应内核函数执行；恢复寄存器；返回用户态
```
$ disassemble $pc, $pc+0xd3
Dump of assembler code from 0xffffffff8177af90 to 0xffffffff8177b063:
=>[system_call+0]:  swapgs                            //保存用户态GS，设置内核GS      
  [system_call+3]:  mov    QWORD PTR gs:0xb000,rsp    //保存用户态栈指针
  [system_call+12]: mov    rsp,QWORD PTR gs:0xb788    //设置内核态栈指针
  [system_call+21]: sti    
  [system_call+22]: data32 data32 xchg ax,ax
  [system_call+26]: data32 xchg ax,ax
  [system_call+29]: sub    rsp,0x50                   //还记得asmlinkage么，将参数复制到栈上
  [system_call+33]: mov    QWORD PTR [rsp+0x40],rdi
  [system_call+38]: mov    QWORD PTR [rsp+0x38],rsi
  [system_call+43]: mov    QWORD PTR [rsp+0x30],rdx
  [system_call+48]: mov    QWORD PTR [rsp+0x20],0xffffffffffffffda
  [system_call+57]: mov    QWORD PTR [rsp+0x18],r8
  [system_call+62]: mov    QWORD PTR [rsp+0x10],r9
  [system_call+67]: mov    QWORD PTR [rsp+0x8],r10
  [system_call+72]: mov    QWORD PTR [rsp],r11
  [system_call+76]: mov    QWORD PTR [rsp+0x48],rax
  [system_call+81]: mov    QWORD PTR [rsp+0x50],rcx
  [system_call+86]: test   DWORD PTR [rsp-0x3f78],0x100801d1
  [system_call+97]: jne    0xffffffff8177b0e2 [tracesys]
  [system_call_fastpath+0]: and    eax,0xbfffffff
  [system_call_fastpath+5]: cmp    eax,0x220
  [system_call_fastpath+10]:        ja     0xffffffff8177b012 [ret_from_sys_call]
  [system_call_fastpath+12]:        mov    rcx,r10
  [system_call_fastpath+15]:        call   QWORD PTR [rax*8-0x7e7feba0] //从sys_call_table取出内核函数指针
  [system_call_fastpath+22]:        mov    QWORD PTR [rsp+0x20],rax     //保存返回值
  [ret_from_sys_call+0]:    mov    edi,0x1008feff
  [sysret_check+0]: cli    
  [sysret_check+1]: data32 data32 xchg ax,ax
  [sysret_check+5]: data32 xchg ax,ax
  [sysret_check+8]: mov    edx,DWORD PTR [rsp-0x3f78]
  [sysret_check+15]:        and    edx,edi
  [sysret_check+17]:        jne    0xffffffff8177b065 [sysret_careful]
  [sysret_check+19]:        mov    rcx,QWORD PTR [rsp+0x50]
  [sysret_check+24]:        mov    r11,QWORD PTR [rsp]
  [sysret_check+28]:        mov    r10,QWORD PTR [rsp+0x8]
  [sysret_check+33]:        mov    r9,QWORD PTR [rsp+0x10]
  [sysret_check+38]:        mov    r8,QWORD PTR [rsp+0x18]
  [sysret_check+43]:        mov    rax,QWORD PTR [rsp+0x20]             //rax保存返回值
  [sysret_check+48]:        mov    rdx,QWORD PTR [rsp+0x30]
  [sysret_check+53]:        mov    rsi,QWORD PTR [rsp+0x38]
  [sysret_check+58]:        mov    rdi,QWORD PTR [rsp+0x40]
  [sysret_check+63]:        mov    rsp,QWORD PTR gs:0xb000              //恢复用户态栈指针
  [sysret_check+72]:        swapgs                                      //恢复用户态GS
  [sysret_check+75]:        rex.W sysret                                //回到用户态
```
3）hello_syscall -> syscall@libc -> system_call(kernel) -> write -> vfs_write:
fs/read_write.c

```
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
                size_t, count)
{
        return **ksys_write**(fd, buf, count);
}

ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
        struct fd f = fdget_pos(fd);
        ssize_t ret = -EBADF;

        if (f.file) {
                loff_t pos, *ppos = file_ppos(f.file);
                if (ppos) {
                        pos = *ppos;
                        ppos = &pos;
                }
                ret = **vfs_write**(f.file, buf, count, ppos);
                if (ret >= 0 && ppos)
                        f.file->f_pos = pos;
                fdput_pos(f);
        }

        return ret;
}
```

4）hello_syscall -> syscall@libc -> system_call(kernel) -> write -> **vfs_write(kernel)** -> **file->f_op->write(driver/module)**
```
static ssize_t __vfs_write(struct file *file, const char __user *p,
                           size_t count, loff_t *pos)
{
        if (file->f_op->write)
                return **file->f_op->write**(file, p, count, pos);
        else if (file->f_op->write_iter)
                return new_sync_write(file, p, count, pos);
        else
                return -EINVAL;
}


```

比如 shell ->execve@libc -> execve
```
$ objdump -T /bin/bash|grep execv
0000000000000000      DF *UND*	0000000000000000  GLIBC_2.2.5 execve   //use
0000000000045010 g    DF .text	0000000000000547  Base        shell_execve


$ objdump -T libc.so|grep execve
00000000000f8860 g    DF .text	0000000000000105  GLIBC_2.2.5 fexecve
0000000000037240  w   DF .text	0000000000000021  GLIBC_2.2.5 execve   //define
```

## 4 自动分析思路讨论

hello_syscall -> syscall@libc -> system_call(kernel) -> write -> vfs_write**(kernel)** -> file->f_op->write**(driver/module)**

需要完成一下几个方面的工作：
- 分析used system calls: 
- 识别kernel中每个system call对应的代码，并移除unused system calls

### 4.1 分析used sytem calls

1) hello_syscall -> syscall@libc

- 二进制级别： 可以通过符号表的reference->define的分析识别，跟libc的分析思路一致
- 源代码级别：函数调用分析可识别

```
$ objdump -d hello_syscall.exe

000000000000068a <main>:
 68a:	55                   	push   %rbp
 68b:	48 89 e5             	mov    %rsp,%rbp
 68e:	b8 00 00 00 00       	mov    $0x0,%eax
 693:	e8 b8 fe ff ff       	callq  550 <getchar@plt>
 698:	b9 0e 00 00 00       	mov    $0xe,%ecx
 69d:	48 8d 15 a0 00 00 00 	lea    0xa0(%rip),%rdx        # 744 <_IO_stdin_used+0x4>
 6a4:	be 01 00 00 00       	mov    $0x1,%esi
 6a9:	bf 01 00 00 00       	mov    $0x1,%edi    // syscall number
 6ae:	b8 00 00 00 00       	mov    $0x0,%eax
 6b3:	e8 a8 fe ff ff       	**callq  560 <syscall@plt>**
 6b8:	b8 00 00 00 00       	mov    $0x0,%eax
 6bd:	5d                   	pop    %rbp
 6be:	c3                   	retq   
 6bf:	90                   	nop

```

```shell
$ readelf -s misc/syscall.os


Symbol table '.symtab' contains 14 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    8 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT   10 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT   13 
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT   11 
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT   14 
    10: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
    11: 0000000000000000    51 FUNC    GLOBAL DEFAULT    1 syscall
    12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    13: 0000000000000000     0 TLS     GLOBAL DEFAULT  UND __libc_errno
```

2) syscall@libc -> system_call(kernel)

- 2.1）二进制级别：syscall（kernel）是指令而不是符号。因为大部分的syscall不需要经过glibc中的syscall函数wrapper对syscall指令的调用(），所以无法通过符号表或者重定位表分析；需要到指令级别信息来分析。

glibc中的syscall函数：

```shell
$ less /glibc-2.31//sysdeps/unix/sysv/linux/x86_64/syscall.S
.text
ENTRY (syscall)
        movq %rdi, %rax         /* Syscall number -> rax.  */
        movq %rsi, %rdi         /* shift arg1 - arg5.  */
        movq %rdx, %rsi
        movq %rcx, %rdx
        movq %r8, %r10
        movq %r9, %r8
        movq 8(%rsp),%r9        /* arg6 is on the stack.  */
        syscall                 /* Do the system call.  */
        cmpq $-4095, %rax       /* Check %rax for error.  */
        jae SYSCALL_ERROR_LABEL /* Jump to error handler if error.  */
        ret                     /* Return to caller.  */

PSEUDO_END (syscall)
```

malloc方法中调用mmap system call的路径，不需要通过libc的syscall wrapper：

$ disassemble __mmap @ sysdeps/unix/sysv/linux/mmap64.c:59  // called by malloc

C: return (void *) MMAP_CALL (mmap, addr, len, prot, flags, fd, offset); @mmap64.c:59

==> return __syscall_map(addr, len, ...)  直接内联成syscall指令，而不是syscall.S中的函数

```shell
assembly: 
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

 - 2.2）二进制反汇编：将obj文件反汇编后，识别出syscall指令以及syscall number。需要跨越多条指令，识别syscall之前最近的%rax/%eax的值，可以通过正则模式匹配；<font color="red">但是如果syscall number的传递，经过更长距离（估计比较少见），就会很慢；也许容易出现错误匹配.***（是否值得尝试？）***</font>

 - 2.3）源代码级别：

- 对于hello_syscall： syscall(SYS_write, 1, "hello, world!\n", 14); 通过分析app代码中syscall第一个参数，提取syscall number

- 对于getchar（）和malloc（）： 因为没有通过libc的syscall wrapper，需要分析inline assembly code，与2.2）类似，也面临类似的挑战

- 2.4）<font color="blue">初步结论：识别used syscall，需要进行汇编分析</font>
  
### 4.2 分析kernel中的system call对应的代码块 （kernel config貌似不能指定不编译的syscall number）
- 1）kernel开启-ffunction重新编译：以便分离不同的系统调用
- 2）根据arch/x86/syscalls/syscall_64.tbl，找到所有的syscall number，以及对应的函数名；结合4.1分析，得到无用的syscall及对应的函数名
- 3）根据函数名识别对应的函数符号（可以二进制级别处理），以及对应的section
- 4）通过修改link script文件，discard这些无用的section
- 5）修改arch/x86/syscalls/syscall_64.tbl，这样kernel init时，会重新生成跳转表


## 5 其他

### 5.1 其他entry

- 1）app入口（一般通过libc等库函数），上述主要讨论

- 2）中断入口（中断向量）

- 3）动态加载的模块

- 4）内核启动入口

### 5.2 syscall是否是代码尺寸的主要源？

```shell
$ du -h --max-depth=1

$ find . -name "*.c"|xargs grep "SYSCALL_DEFINE"
```