---
layout: default
title: initcall机制详解
nav_order: 4
parent: Linux内核
author: plot
---
{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>

在Linux内核开发的早期阶段就实现了initcall，旨用于在启动过程调用函数。在2018年Steven Robstedt添加了trace支持，用于测量每个initcall所花费的时间，以便调试。

initcall背后的基本思想如下：
* 使用initcall将在的对象文件中创建ELF节
* 程序员创建所需的函数，放入这些创建的ELF节
* 内核在启动时便利这些ELF节，执行其中的函数
* initcall有不同的级别，用于组织函数

使用 initcall 有以下好处：代码更模块化、更易维护，因为我们不需要明确地传递、存储和调用函数指针，相反，我们只需将函数标记为适当级别的 initcall，它就会在启动阶段的合适时间自动调用。然而，要知道应该使用哪一个是很难区分的。级别名称反映了 initcall 的顺序，即 initcall 将在哪个部分被调用，所以决定函数级别时，必须了解函数的依赖关系以及函数应在何处执行。

有时并不需要使用 initcall，使用 module_init()就足够了。请注意，initcall 函数只能用于静态内置模块。对于动态可装载的模块，必须使用module_init()。

### initcall实现

initcall 的一部分在 include/linux/init.h 中实现。

```
#define pure_initcall(fn)			__define_initcall(fn, 0)
#define core_initcall(fn)			__define_initcall(fn, 1)
#define postcore_initcall(fn)			__define_initcall(fn, 2)
#define arch_initcall(fn)			__define_initcall(fn, 3)
#define subsys_initcall(fn)			__define_initcall(fn, 4)
#define fs_initcall(fn)				__define_initcall(fn, 5)
#define rootfs_initcall(fn)			__define_initcall(fn, rootfs)
#define device_initcall(fn)			__define_initcall(fn, 6)
#define late_initcall(fn)			__define_initcall(fn, 7)
```
通过带有两个参数的通用 \_\_define_initcall() 来定义 initcall：

- 函数名
- 一个 ID，用于对 initcall 进行排序。ID既可以是数字，也可以是字符串，参见 rootfs。不同类型的 initcall 会有不同的 ID。
（rootfs级别的函数主要用于挂载 rootfs，可以从 initramfs 或块设备挂载。）

```
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

#define ___define_initcall(fn, id, __sec) \
static initcall_t __initcall_##fn##id __used \
__attribute__((__section__(#__sec ".init"))) = fn;
```

\_\_\_define_initcall()宏使用以下的参数：
* initcall函数名称
* initcall ID
* 创建section节的名称

以`core_initcall(plot)`为例，看一下宏生成的代码：
```
core_initcall(plot)
-->__define_initcall(plot,1)
	-->___define_initcall(plot,1,.initcall1)
	static initcall_t __initcall_plot1 __used \
	__attribute__((__section__("initcall1.init"))) = plot;
```
所有这些参数都将用于创建一个 initcall_t 条目，该条目将根据给定的参数命名。在例子中，是 \_\_initcall_plot1。它指向函数plot。然后通过使用关键字attribute和section，将\_\_initcall_plot1放入.initcall1.init段中。

.initcall1.init段包含所有已注册core initcall的函数地址。其顺序在编译时确定，取决于Makefile。

每种initcall都有一个ID，所以每种类型的initcall根据其ID有不同的段名：.initcall1.init，.initcall2.init。

initcall 排序的主要实现是在 init/main.c 中完成的。

initcall_levels 是一个数组，其中每个条目都是该特定级别的指针。

```
extern initcall_entry_t __initcall_start[];
extern initcall_entry_t __initcall0_start[];
extern initcall_entry_t __initcall1_start[];
extern initcall_entry_t __initcall2_start[];
extern initcall_entry_t __initcall3_start[];
extern initcall_entry_t __initcall4_start[];
extern initcall_entry_t __initcall5_start[];
extern initcall_entry_t __initcall6_start[];
extern initcall_entry_t __initcall7_start[];
extern initcall_entry_t __initcall_end[];

static initcall_entry_t *initcall_levels[] __initdata = {
        __initcall0_start,
        __initcall1_start,
        __initcall2_start,
        __initcall3_start,
        __initcall4_start,
        __initcall5_start,
        __initcall6_start,
        __initcall7_start,
        __initcall_end,
};
```
我们已经知道，initcall 是一种将选定函数放在特定段中的机制。这些函数将在启动时被遍历。为此，内核必须以某种方式知道它们的实际位置。这可以通过链接器使用脚本来实现，脚本会创建 \_\_initcall_start 符号（include/asm-generic/vmlinux.lds.h）：

```
#define INIT_CALLS_LEVEL(level)                   \
                __initcall##level##_start = .;    \
                KEEP(*(.initcall##level##.init))  \
                KEEP(*(.initcall##level##s.init)) \
```

编译后，生成的链接器脚本（arch/arm/kernel/vmlinux.lds）看起来像这样：

```
.init.data : AT(ADDR(.init.data) - 0)

__initcall_start = .; 			KEEP(*(.initcallearly.init))
__initcall0_start = .; 		KEEP(*(.initcall0.init))
__initcall1_start = .; 		KEEP(*(.initcall1.init))
__initcall2_start = .; 		KEEP(*(.initcall2.init))
__initcall3_start = .; 		KEEP(*(.initcall3.init))
__initcall4_start = .; 		KEEP(*(.initcall4.init))
__initcall5_start = .; 		KEEP(*(.initcall5.init))
__initcallrootfs_start = .; 	KEEP(*(.initcallrootfs.init))
__initcall6_start = .; 		KEEP(*(.initcall6.init))
__initcall7_start = .; 		KEEP(*(.initcall7.init))
__initcall_end = .
```

处理所有可能的 initcall 级别的主函数名为 do_initcalls()，位于 init/main.c 中：

```
static void __init do_basic_setup(void)
{
	[...]
	do_initcalls();
}

static void __init do_initcalls(void)
{
	int level;
	[...]

	for (level = 0; level < ARRAY_SIZE(initcall_levels)–1;level++) {
		[...]
		do_initcall_level(level, command_line);
	}
}
```

该函数将处理数组中的所有级别。关于命令行参数（command_line parameter），它只是通常命令行的一个副本，可以包含模块参数。该函数正在调用另一个函数 do_initcall_level，其代码（简化）如下：

```
static void __init do_initcall_level(int level,char *command_line)
{
	initcall_entry_t *fn;
	[...]
	for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(initcall_from_entry(fn));
}
```
do_initcall_level通过 do_one_initcall 函数调用特定级别的所有 initcall。

```
int __init_or_module do_one_initcall(initcall_t fn) {
		int ret;
		[...]

		do_trace_initcall_start(fn);
		ret = fn();
		do_trace_initcall_finish(fn, ret);
		[...]

		return ret;
}
```
上述代码有两个要点：  
* 使用开始/结束跟踪函数（参见调试部分） 
* 执行与用户创建的函数相对应的 initcall_t

总而言之，initcall_levels 是一个数组，包含所有initcall级别的 initcall\<n\>\_start 列表。initcall\<n\>\_start对应于该级别的第一个.initcall\<n\>.init部分。而.initcall\<n\>.init存储的是initcall_t条目，它是一个函数指针，指向预设的函数。最终执行通过core_initcall()等宏修饰的函数。


### 调试

调试命令行参数用于在执行所有 initcalls 函数时打印 2 条信息。这是一个很好的调试参数，可用于检测哪些 initcall 占用了太多时间，尤其是在改进启动时间的情况下。详见参考链接。

### 参考文献

[An introduction to Linux kernel initcalls](https://www.collabora.com/news-and-blog/blog/2020/07/14/introduction-to-linux-kernel-initcalls/)

[Initcalls, part 2: Digging into implementation](https://www.collabora.com/news-and-blog/blog/2020/09/25/initcalls-part-2-digging-into-implementation/)