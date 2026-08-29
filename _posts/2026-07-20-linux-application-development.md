---
layout: post
title: Linux 应用开发学习笔记
date: 2026-07-20 19:31:33 +0800
description: 整理 Linux 应用开发中的 GCC、Makefile、文件 I/O、Framebuffer、输入子系统与 select/poll 编程基础。
tags: [Linux, 应用开发, GCC, Makefile]
categories: [Linux学习]
published: true
toc:
  sidebar: left
---

## 应用开发基础

### GCC编译

#### 编译过程分析

一个 C/C++文件要经过预处理(preprocessing)、编译(compilation)、汇编(assembly)和链接(linking)等 4 步才能变成可执行文件 。

我们使用的代码是`gcc -o bin hello.c`，实际上的过程如下：

![GCC 编译流程]({{ '/assets/img/blog/linux-application-development/gcc-compilation-pipeline.png' | relative_url }})

使用`gcc -o bin hello.c -v`观察详细过程如下：

```text
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/aarch64-linux-gnu/13/lto-wrapper
OFFLOAD_TARGET_NAMES=nvptx-none
OFFLOAD_TARGET_DEFAULT=1
Target: aarch64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 13.3.0-6ubuntu2~24.04.1' --with-bugurl=file:///usr/share/doc/gcc-13/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-13 --program-prefix=aarch64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/libexec --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-libstdcxx-backtrace --enable-gnu-unique-object --disable-libquadmath --disable-libquadmath-support --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --enable-fix-cortex-a53-843419 --disable-werror --enable-offload-targets=nvptx-none=/build/gcc-13-f9pAEi/gcc-13-13.3.0/debian/tmp-nvptx/usr --enable-offload-defaulted --without-cuda-driver --enable-checking=release --build=aarch64-linux-gnu --host=aarch64-linux-gnu --target=aarch64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=4
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1)
COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin-'
 /usr/libexec/gcc/aarch64-linux-gnu/13/cc1 -quiet -v -imultiarch aarch64-linux-gnu hello.c -quiet -dumpdir bin- -dumpbase hello.c -dumpbase-ext .c -mlittle-endian -mabi=lp64 -version -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -o /tmp/ccCPUi5x.s
GNU C17 (Ubuntu 13.3.0-6ubuntu2~24.04.1) version 13.3.0 (aarch64-linux-gnu)
        compiled by GNU C version 13.3.0, GMP version 6.3.0, MPFR version 4.2.1, MPC version 1.3.1, isl version isl-0.26-GMP

GGC heuristics: --param ggc-min-expand=100 --param ggc-min-heapsize=131072
ignoring nonexistent directory "/usr/local/include/aarch64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/aarch64-linux-gnu/13/include-fixed/aarch64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/aarch64-linux-gnu/13/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/aarch64-linux-gnu/13/../../../../aarch64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/aarch64-linux-gnu/13/include
 /usr/local/include
 /usr/include/aarch64-linux-gnu
 /usr/include
End of search list.
Compiler executable checksum: af3ce90fa71916e7a8489f94b60f417c
COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin-'
 as -v -EL -mabi=lp64 -o /tmp/cc6QSwBH.o /tmp/ccCPUi5x.s
GNU assembler version 2.42 (aarch64-linux-gnu) using BFD version (GNU Binutils for Ubuntu) 2.42
COMPILER_PATH=/usr/libexec/gcc/aarch64-linux-gnu/13/:/usr/libexec/gcc/aarch64-linux-gnu/13/:/usr/libexec/gcc/aarch64-linux-gnu/:/usr/lib/gcc/aarch64-linux-gnu/13/:/usr/lib/gcc/aarch64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/aarch64-linux-gnu/13/:/usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/:/usr/lib/gcc/aarch64-linux-gnu/13/../../../../lib/:/lib/aarch64-linux-gnu/:/lib/../lib/:/usr/lib/aarch64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/aarch64-linux-gnu/13/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin.'
 /usr/libexec/gcc/aarch64-linux-gnu/13/collect2 -plugin /usr/libexec/gcc/aarch64-linux-gnu/13/liblto_plugin.so -plugin-opt=/usr/libexec/gcc/aarch64-linux-gnu/13/lto-wrapper -plugin-opt=-fresolution=/tmp/cc5vlda9.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --eh-frame-hdr --hash-style=gnu --as-needed -dynamic-linker /lib/ld-linux-aarch64.so.1 -X -EL -maarch64linux --fix-cortex-a53-843419 -pie -z now -z relro -o bin /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/Scrt1.o /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/crti.o /usr/lib/gcc/aarch64-linux-gnu/13/crtbeginS.o -L/usr/lib/gcc/aarch64-linux-gnu/13 -L/usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu -L/usr/lib/gcc/aarch64-linux-gnu/13/../../../../lib -L/lib/aarch64-linux-gnu -L/lib/../lib -L/usr/lib/aarch64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/aarch64-linux-gnu/13/../../.. /tmp/cc6QSwBH.o -lgcc --push-state --as-needed -lgcc_s --pop-state -lc -lgcc --push-state --as-needed -lgcc_s --pop-state /usr/lib/gcc/aarch64-linux-gnu/13/crtendS.o /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/crtn.o
COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin.'
```

关键点：

预处理：把.c转换成.i，编译成.s，关键词为`cc1`

` /usr/libexec/gcc/aarch64-linux-gnu/13/cc1 -quiet -v -imultiarch aarch64-linux-gnu hello.c  -o /tmp/ccCPUi5x.s`

汇编：把.s转换成.o，关键词为`as`

` as -v -EL -mabi=lp64 -o /tmp/cc6QSwBH.o /tmp/ccCPUi5x.s`

链接：把一系列.o链接成一个可执行文件app，关键词为`collect`

`COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin.'
 /usr/libexec/gcc/aarch64-linux-gnu/13/collect2 -plugin /usr/libexec/gcc/aarch64-linux-gnu/13/liblto_plugin.so -plugin-opt=/usr/libexec/gcc/aarch64-linux-gnu/13/lto-wrapper -plugin-opt=-fresolution=/tmp/cc5vlda9.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --eh-frame-hdr --hash-style=gnu --as-needed -dynamic-linker /lib/ld-linux-aarch64.so.1 -X -EL -maarch64linux --fix-cortex-a53-843419 -pie -z now -z relro -o bin /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/Scrt1.o /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/crti.o /usr/lib/gcc/aarch64-linux-gnu/13/crtbeginS.o -L/usr/lib/gcc/aarch64-linux-gnu/13 -L/usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu -L/usr/lib/gcc/aarch64-linux-gnu/13/../../../../lib -L/lib/aarch64-linux-gnu -L/lib/../lib -L/usr/lib/aarch64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/aarch64-linux-gnu/13/../../.. /tmp/cc6QSwBH.o -lgcc --push-state --as-needed -lgcc_s --pop-state -lc -lgcc --push-state --as-needed -lgcc_s --pop-state /usr/lib/gcc/aarch64-linux-gnu/13/crtendS.o /usr/lib/gcc/aarch64-linux-gnu/13/../../../aarch64-linux-gnu/crtn.o
COLLECT_GCC_OPTIONS='-o' 'bin' '-v' '-mlittle-endian' '-mabi=lp64' '-dumpdir' 'bin.' `

![GCC 详细编译输出]({{ '/assets/img/blog/linux-application-development/gcc-verbose-output.png' | relative_url }})

#### gcc常用命令

| 常用选项 | 描述                                               |
| -------- | -------------------------------------------------- |
| -E       | 预处理，开发过程中想快速确定某个宏可以使用“-E -dM” |
| -c       | 把预处理、编译、汇编都做了，但是不链接             |
| -o       | 指定输出文件                                       |
| `-I`     | 指定头文件目录                                     |
| -L       | 指定链接时库文件目录                               |
| `-l`     | 指定链接哪一个库文件                               |

针对头文件，可以用`< >`来指定的目录去查找，也可以用`-I` 指定头文件目录；

针对库文件，可以用`-l`来使用指定链接的库文件，也可以使用`-L` 指定链接库文件目录；

> ① 怎么确定交叉编译器中头文件的默认路径？
>
> 进入交叉编译器的目录里，执行： find -name “stdio.h”，它位于一个“include”目录下的根目录里。这个“include”目录，就是要找的路径。
>
> ② 怎么自己指定头文件目录？
>
> 编译时，加上“-I <头文件目录>”这样的选项。
>
> ③ 怎么确定交叉编译器中库文件的默认路径？
>
> 进入交叉编译器的目录里，执行： find -name lib，可以得到 xxxx/lib、 xxxx/usr/lib，一般来说这 2 个目录就是要找的路径。
>
> 如果有很多类似的 lib，进去看看，有很多 so 文件的目录一般就是要找的路径。
>
> ④ 怎么自己指定库文件目录、指定要用的库文件？
>
> 编译时，加上“-L <库文件目录>”这样的选项，用来指定库目录；编译时，加上“-labc”这样的选项，用来指定库文件 libabc.so。

库可以分为**静态库**和**动态库**，使用方法不一样

### Makefile

常用形式如下：

```text
目标(target)…: 依赖(prerequiries)…
<tab>命令(command)
```

如果“依赖文件”比“目标文件”更加新，那么执行“命令”来重新生成“目标文件”。命令被执行的 2 个条件：依赖文件比目标文件新，或是 目标文件还没生成。

#### 变量的用法

```text
A = xxx  // 延时变量
B ?= xxx // 延时变量，只有第一次定义时赋值才成功；如果曾定义过，此赋值无效
C := xxx // 立即变量
D += yyy // 如果 D 在前面是延时变量，那么现在它还是延时变量；
         // 如果 D 在前面是立即变量，那么现在它还是立即变量
```

#### 常用函数

1. `$(foreach var,list,text)`
   简单地说，就是 for each var in list, change it to text。
   对 list 中的每一个元素，取出来赋给 var，然后把 var 改为 text 所描述的形式。例如：

```text
objs := a.o b.o
dep_files := $(foreach f, $(objs), .$(f).d) // 最终 dep_files := .a.o.d .b.o.d
```

2. `$(wildcard pattern)`
   pattern 所列出的文件是否存在，把存在的文件都列出来

```text
src_files := $( wildcard *.c) // 最终 src_files 中列出了当前目录下的所有.c 文件
```

3. `$(filter pattern...,text)`
   把 text 中符合 pattern 格式的内容， filter(过滤)出来、留下来。
   `$(filter-out pattern...,text)`
   把 text 中符合 pattern 格式的内容， filter-out(过滤)出来、扔掉。

```makefile
obj-y := a.o b.o c/ d/
DIR := $(filter %/, $(obj-y)) //结果为： c/ d/
obj-y := a.o b.o c/ d/
DIR := $(filter-out %/, $(obj-y)) //结果为： a.o b.o
```

4.`$(patsubst pattern, replacement, text)`
寻找`text`中符合格式`pattern`的字，用`replacement`替换它们。 `pattern`和`replacement`中可以使用通配符。
比如：

```text
subdir-y := c/ d/
subdir-y := $(patsubst %/, %, $(subdir-y)) // 结果为： c d
```

#### Makefile实例

```text
gcc -M c.c // 打印出依赖
gcc -M -MF c.d c.c // 把依赖写入文件c.d
gcc -c -o c.o c.c -MD -MF c.d // 编译c.o, 把依赖写入文件c.d
```

## 文件操作

linux系统中，**一切皆文件**，所有的操作，都是通过**文件 IO**来操作的。

### 文件IO

分为**标准IO**和**系统调用IO**，系统调用IO是针对内核写的，直接操作系统内核，缺点是调用一次就会进入一次内核；标准IO在系统调用IO的基础上进行封装，可以通过编译后运行在不同的unix-like系统和windows下，而且引入了用户buffer，不需要频繁访问内核。
![标准 I/O 与系统调用 I/O 层次]({{ '/assets/img/blog/linux-application-development/io-layering.png' | relative_url }})

- 系统调用 I/O：`open/read/write/lseek/close`
- 标准 I/O：`fopen/fread/fwrite/flseek/fclose`

#### open和close

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
//关闭已打开的文件
int close(int fd);
```

具体函数使用查询手册，不在此展开。

#### read和write

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

#### dup和dup2

![dup 与文件描述符表关系]({{ '/assets/img/blog/linux-application-development/dup-file-table.png' | relative_url }})

```c
#include <unistd.h>
int dup(int oldfd);
int dup2(int oldfd, int newfd);
```

针对linux来说，一个fd对应一个file结构体，结构体里有一个`f_pos`来对应当前访问到的位置，如果使用open函数打开同一个txt，赋给不同的句柄。

```c
fd = open(argv[1], ...);
fd2 = open(argv[1], ...);
```

即使打开的是同一个文件1.txt，但是他们对应的句柄不同，使用`read`函数时的`f_pos`也不同。
此时使用`dup`

```c
fd3 = dup(fd);
```

则fd与fd3使用的是同一个file结构体，使用read之后，它们的f_pos都会更新。

`dup2`函数是`dup`的更近一步，它的执行过程是：

1. 如果 newfd 已经打开，dup2 会先将其关闭。
2. 将 oldfd 复制给 newfd，使两者指向同一个文件表项（共享文件偏移量、状态标志等）。
3. 如果 oldfd 与 newfd 相等，则 dup2 什么都不做，直接返回 newfd。

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char **argv){
    int fd, len;
    if(argc != 2){
        printf("Usage: %s <file> \r\n", argv[0]);
        return -1;
    }
    fd = open(argv[1], O_RDWR, 0666);
    if(fd < 0){
        perror("open");
    }
    else {
        printf("fd = %d \r\n", fd);
    }
    dup2(fd, 1);
    printf("hello world!\n");
    return 0;
}
// gcc -o dup2 dup2.c
// ./dup2 1.txt
// 现象为1.txt里的123 -> hello world!
```

## 字符显示

### Framebuffer

frame表示帧,buffer表示缓存,framebuffer里保存着一帧图像里的每一个像素点的颜色值.

#### LCD的操作原理:

1. 驱动程序设置好LCD控制器：
   根据LCD的参数设置LCD控制器的时序、信号极性；
   根据LCD分辨率、 BPP分配Framebuffer。
2. APP 使用ioctl获得LCD分辨率、 BPP
3. APP 通过 mmap 映射 Framebuffer，在 Framebuffer 中写入数据

![Framebuffer 应用访问流程]({{ '/assets/img/blog/linux-application-development/framebuffer-workflow.png' | relative_url }})

![Framebuffer 像素内存布局]({{ '/assets/img/blog/linux-application-development/framebuffer-memory-layout.png' | relative_url }})
假设`fb_base`为mmap映射出的framebuffer内存地址,则`(x,y)`的地址计算公式为:
`(x,y)起始地址 = fb_base + (xres*bpp/8) * y + x*bpp/8`

#### 像素的颜色表示bpp

像素的颜色使用RGB来表示,不同的BPP格式里使用不同位数的RGB,分为32BPP,24BPP,16BPP

![RGB 像素格式]({{ '/assets/img/blog/linux-application-development/rgb-pixel-formats.png' | relative_url }})
针对32BPP，高八位为透明度，不考虑。
24BPP，对于 24BPP，硬件上为了方便处理，在 Framebuffer 中也是用32 位来表示，效果跟32BPP是一样的。
针对16BPP，可分为RGB565和RGB555.根据RGB565的格式，只保留red中的**高 5 位**、green中的**高 6 位**、blue中的**高 5 位**，组合成一个新的 16 位颜色值。

### 字符编码

## 输入系统

驱动程序上报的数据含义的三项重要内容：
type：哪类？ 例如EV_KEY，按键类
code：哪个？ 例如KEY_A
value：值 0-松开 1-按下 2-长按
除此之外，还上报一个运行时间，每个输入事件 input_event 中都含有发生时间：timeval 表示的是“自系统启动以来过了多少时间”，它是一个结构体，含有“ tv_sec、 tv_usec”两项(即秒、微秒)。

```c
/*
 * The event structure itself
 * Note that __USE_TIME_BITS64 is defined by libc based on
 * application's request to use 64 bit time_t.
 */

struct input_event {
#if (__BITS_PER_LONG != 32 || !defined(__USE_TIME_BITS64)) && !defined(__KERNEL__)
	struct timeval time;
#define input_event_sec time.tv_sec
#define input_event_usec time.tv_usec
#else
	__kernel_ulong_t __sec;
#if defined(__sparc__) && defined(__arch64__)
	unsigned int __usec;
	unsigned int __pad;
#else
	__kernel_ulong_t __usec;
#endif
#define input_event_sec  __sec
#define input_event_usec __usec
#endif
	__u16 type;
	__u16 code;
	__s32 value;
};
```

### type、code和value

**type**：表示哪类事件，比如 EV_KEY 表示按键类、 EV_REL表示相对位移(比如鼠标)， EV_ABS 表示绝对位置(比如触摸屏),如下：

```c
/*
 * Event types
 */
#define EV_SYN			0x00
#define EV_KEY			0x01
#define EV_REL			0x02
#define EV_ABS			0x03
#define EV_MSC			0x04
#define EV_SW			0x05
#define EV_LED			0x11
#define EV_SND			0x12
#define EV_REP			0x14
#define EV_FF			0x15
#define EV_PWR			0x16
#define EV_FF_STATUS		0x17
#define EV_MAX			0x1f
#define EV_CNT			(EV_MAX+1)
```

**code**：表示该类事件下的哪一个事件，比如对于EV_KEY(按键)类事件，它表示键盘。键盘上有很多按键，比如数字键 1、 2、 3，字母键 A、 B、 C 里等。所以可以这些事件：

```c
/*
 * Keys and buttons
 *
 * Most of the keys/buttons are modeled after USB HUT 1.12
 * (see http://www.usb.org/developers/hidpage).
 * Abbreviations in the comments:
 * AC - Application Control
 * AL - Application Launch Button
 * SC - System Control
 */
#define KEY_RESERVED		0
#define KEY_ESC			1
#define KEY_1			2
#define KEY_2			3
#define KEY_3			4
#define KEY_4			5
#define KEY_5			6
#define KEY_6			7
#define KEY_7			8
#define KEY_8			9
#define KEY_9			10
#define KEY_0			11
#define KEY_MINUS		12
#define KEY_EQUAL		13
#define KEY_BACKSPACE		14
#define KEY_TAB			15
#define KEY_Q			16
··· //不在此一一列举，具体查看/usr/include/linux/input-event-codes.h
```

对于触摸屏，它提供的是绝对位置信息，有 X 方向、 Y 方向，还有压力值。

```c
/*
 * Absolute axes
 */

#define ABS_X			0x00
#define ABS_Y			0x01
#define ABS_Z			0x02
```

**value**：表示事件值，对于按键，它的 value 可以是 0(表示按键被按下)、 1(表示按键被松开)、2(表示长按)；对于触摸屏，它的 value 就是坐标值、压力值。

**事件之间的界限：** type、code、value值全未0，表示一个读完一个同步事件的所有数据。

**事件举例：**

```c
hexdump /dev/input/mouse0
0000000 0108 0800 0001 0108 0800 0001 0108 0800
0000010 0001 0108 0800 0001 0108 0800 0001 0108
0000020 0800 0001 0108 0800 0001 0108 0800 0001
0000030 0108 0800 0001 0108 2800 ff00 0108 0800
0000040 0001 0108 0800 0001 0008 0801 0001 0108
0000050 0800 0001 0108 0800 0001 0008 1801 00ff
0000060 ff18 1800 00ff ff18 1800 00ff ff18 1800
0000070 00ff ff18 1800 00ff ff18 1800 00ff ff18
0000080 1801 00ff ff18 1800 00fe ff18 1800 01ff
0000090 ff18 1800 00ff ff18 1800 00ff ff18 1800
00000a0 00ff fe18 1800 00ff ff18 1800 00ff ff18
00000b0 1800 00ff ff18 1800 00ff ff18 0900 0000
00000c0 0008 0a00 0000 0008 0900 0000 0008 0800
00000d0 0100 0108 0801 0101 0008 0801 0101 0108
00000e0 0801 0101 0108 0801 0001 0208 0800 0001
00000f0 0108 0800 0002 0128 08ff 0001 0108 0800
0000100 0002 0208 0800 0001 0208 0800 0001 0108
0000110 0800 0001 0208 0800 0001 0108 0800 0001
//序号      秒       微秒    type code   value
```

### 调试技巧

- 查看所有输入设备节点名

```shell
ls /dev/input/* -l
```

- 查找设备节点对应的硬件

```shell
cat /proc/bus/input/devices
```

![Linux 输入设备能力位图]({{ '/assets/img/blog/linux-application-development/input-device-bitmap.png' | relative_url }})
重点关注 **B:** 位图

举例 `EV = 17` ,二进制为`0001 0111`，查EV表，得出该设备具有`EV_SYN` `EV_KEY` `EV_REL` `EV_MSC` 四类事件。以下位图解析同理。

### 系统的四种机制

#### 查询方式

APP 调用 open 函数时，传入“ O_NONBLOCK”表示“非阻塞”。
APP 调用 read 函数读取数据时，如果驱动程序中有数据，那么 APP 的 read
函数会返回数据，否则也会立刻返回错误。

#### 休眠-唤醒方式

APP 调用 open 函数时，不传入“O_NONBLOCK”。
APP 调用 read 函数读取数据时，如果驱动程序中有数据，那么 APP 的 read函数会返回数据；否则 APP 就会在内核态休眠，当有数据时驱动程序会把 APP唤醒，read 函数恢复执行并返回数据给 APP。

#### POLL/SELECT方式

两种方式机制一样，只是调用接口不一样。

```c
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
//-----------------------------------------------------
#include <sys/select.h>

typedef /* ... */ fd_set;

int select(int nfds, fd_set *_Nullable restrict readfds,
            fd_set *_Nullable restrict writefds,
            fd_set *_Nullable restrict exceptfds,
            struct timeval *_Nullable restrict timeout);

/*
nfds：监控的所有文件描述符中的最大值 加 1（例如你监控 fd=3, 5, 8，则 nfds 为 9）。
readfds：监控“可读”事件的集合。
writefds：监控“可写”事件的集合（通常传 NULL）。
exceptfds：监控“异常”事件的集合（通常传 NULL）。
timeout：超时时间。如果传 NULL 则无限等待；如果时间设为 0 则立即返回（轮询）。
*/

//fd_set 是一个位图（Bitmask），你不能直接操作它，必须使用以下四个宏
void FD_CLR(int fd, fd_set *set); //将一个 fd 移出集合
int  FD_ISSET(int fd, fd_set *set); //检查某个 fd 是否就绪（通常在 select 返回后使用）。
void FD_SET(int fd, fd_set *set); //将一个 fd 加入集合
void FD_ZERO(fd_set *set); //清空集合
```

原理类似于定一个闹钟，作用是：如果驱动内有数据，则立即返回；如果驱动内没有数据，则进入休眠模式，在休眠期间如果驱动内有数据了，就立即唤醒APP，然后返回数据；如果在超时时间内一直没有数据，则返回timeout。
APP可以根据返回值判断返回的原因并进行处理。
poll和select函数可以同时监测多个事件。
![select 与 poll 等待流程]({{ '/assets/img/blog/linux-application-development/select-poll-workflow.png' | relative_url }})

具体实现：

```c
#include <poll.h>

struct pollfd fds[1];
nfds_t nfds = 1; //fd的数量

···

while(1){
    fds[0].fd = fd;
    fds[0].events = POLLIN; //有数据可读
    fds[0].revents = 0;
    ret = poll(fds, nfds, 5000); //5000ms超时时间
    if(ret > 0 ){
        if(fds[0].revents == POLLIN){ //返回有数据可读
            while(read(fds[0].fd, &event, sizeof(event)) == sizeof(event)){
                printf("get event: type= 0x%x, code= 0x%x, value= 0x%x \n", event.type, event.code, event.value);
            }
        }
    }
    else if(ret == 0){
        printf("time out \n");
    }
    else{
        printf("err poll \n");
    }
}
```

**扩展1：使用 poll 函数监测多个输入设备**

```c
while(1){
    fds[0].fd = fd;
    fds[0].events = POLLIN;
    fds[0].revents = 0;
    fds[1].fd = fd1;
    fds[1].events = POLLIN;
    fds[1].revents = 0;
    ret = poll(fds, nfds, 5000);
    if(ret > 0 ){
        if(fds[0].revents & POLLIN){
            while(read(fds[0].fd, &event, sizeof(event)) == sizeof(event)){
                printf("get event0: type= 0x%x, code= 0x%x, value= 0x%x \n", event.type, event.code, event.value);
            }
        }
        if(fds[1].revents & POLLIN){
            while(read(fds[1].fd, &event, sizeof(event)) == sizeof(event)){
                printf("get event6: type= 0x%x, code= 0x%x, value= 0x%x \n", event.type, event.code, event.value);
            }
        }
    }
    else if(ret == 0){
        printf("time out \n");
    }
    else{
        printf("err poll \n");
    }
}
```

**拓展2：使用SELECT检测设备**

```c
/*
select() 的工作方式比较“笨”：你把集合交给内核，内核观察一圈后，把没准备好的 fd 从集合里剔除，然后返回。

准备：清空 fd_set，把需要监控的 fd 全丢进去。

调用：执行 select()，进程进入睡眠。

返回：当有数据时，内核唤醒进程。

轮询检查：你需要用 FD_ISSET 挨个问一遍：“是你准备好了吗？”
*/
int retval;
fd_set rfds;
struct timeval tv;

···

while(1){
    FD_ZERO(&rfds);
    FD_SET(fd, &rfds);
    tv.tv_sec = 5; //5分钟
    tv.tv_usec = 0;

    retval = select(fd+1, &rfds, NULL, NULL, &tv);
    if(retval < 0 ){
        printf("err select \n");
        break;
    }
    else if(retval == 0){
        printf("time out \n");
    }
    else{
        if(FD_ISSET(fd, &rfds)){
            while(read(fd, &event, sizeof(event)) == sizeof(event)){
                printf("get event: type= 0x%x, code= 0x%x, value= 0x%x \n", event.type, event.code, event.value);
            }
        }
    }
}
```

#### 异步方式

所谓同步，就是“你慢我等你”。
那么异步就是：你慢那你就自己玩，我做自己的事去了，有情况再通知我。
所谓异步通知，就是 APP 可以忙自己的事，当驱动程序用数据时它会主动给APP 发信号，这会导致 APP 执行信号处理函数。

流程如下：

```c
//编写信号处理函数：
static void sig_func(int sig)
{
    int val;
    read(fd, &val, 4);
    printf("get button : 0x%x\n", val);
}
//注册信号处理函数：
signal(SIGIO, sig_func);
//打开驱动：
fd = open(argv[1], O_RDWR);
//把进程 ID 告诉驱动：
fcntl(fd, F_SETOWN, getpid());
//使能驱动的 FASYNC 功能：
flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | FASYNC);
```

## 网络系统

TCP和UDP的不同点在于TCP需要进行三次握手，是可靠连接。

### TCP

> [!NOTE]
> 在嵌入式 Linux 环境下进行 TCP 通信，本质上是基于 Socket（套接字）编程。TCP 是一种面向连接的、可靠的传输层协议，通信过程可以概括为“三次握手建立连接 -> 数据传输 -> 四次挥手释放连接”。
>
> TCP 通信分为 服务器端（Server） 和 客户端（Client） 两个角色，其执行逻辑如下：
> 服务器端：创建 Socket -> 绑定地址 (bind) -> 监听 (listen) -> 等待连接 (accept) -> 接收/发送数据 -> 关闭连接。
> 客户端：创建 Socket -> 发起连接 (connect) -> 发送/接收数据 -> 关闭连接。
>
> **server.c**

```c
#include <stdio.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <signal.h>

/*
*  创建 Socket -> 绑定地址 (bind) -> 监听 (listen) -> 等待连接 (accept) -> 接收/发送数据 -> 关闭连接
*/

#define Server_PORT 8888
#define BACKLOG     10


int main(int argc, char **argv){
    int SocketServer;
    int SocketClient;
    int iRet;
    struct sockaddr_in SocketServerAddr;
    struct sockaddr_in SocketClientAddr;
    int iAddrLen;
	int iClientNum = -1;
    unsigned char RecVbuf[1000];
    int iRecvLen;

    signal(SIGCHLD, SIG_IGN);
    //先建立一个socket
    SocketServer = socket(AF_INET, SOCK_STREAM, 0);
    if(SocketServer == -1){
        perror("socket");
        return -1;
    }
    //绑定地址
    SocketServerAddr.sin_family = AF_INET;
    SocketServerAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    SocketServerAddr.sin_port = htons(Server_PORT);
    memset(SocketServerAddr.sin_zero, 0, 8);

    iRet = bind(SocketServer, (const struct sockaddr *)&SocketServerAddr, sizeof(struct sockaddr));
    if(iRet == -1){
        perror("bind");
        return -1;
    }

    //开始监听
    iRet = listen(SocketServer, BACKLOG);
    if(iRet == -1){
        perror("listen");
        return -1;
    }

    while(1){
        iAddrLen  = sizeof(struct sockaddr);
        //等待连接 (accept)
        SocketClient = accept(SocketServer, (struct sockaddr *)&SocketClientAddr, &iAddrLen);
        if(SocketClient != -1){
            iClientNum++;
            printf("Get connect from client %d : %s\n",  iClientNum, inet_ntoa(SocketClientAddr.sin_addr));
            if(!fork()){ //创建子进程
                while(1){
                    iRecvLen = recv(SocketClient, RecVbuf, 999, 0);
                    if(iRecvLen <= 0){
                        perror("recv");
                        close(SocketClient);
                        return -1;
                    }
                    else{
                        RecVbuf[iRecvLen] = '\0';
                        printf("Get Msg From Client %d: %s\n", iClientNum, RecVbuf);
                    }
                }
            }
        }
    }

	close(SocketServer);
	return 0;
}
```

**client.c**

```c
#include <stdio.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

/*
*  创建 Socket -> 发起连接 (connect) -> 发送/接收数据 -> 关闭连接。
*/

#define Server_PORT 8888

int main(int argc, char **argv){
    int iSocketServer;
    int iRet;
    struct sockaddr_in SocketServerAddr;
    unsigned char SendBuf[1000];
    int RecvLen;
    if(argc != 2){
        printf("Usage: %s <ipaddr> \n", argv[0]);
        return -1;
    }

    //创建Socket
    iSocketServer = socket(AF_INET, SOCK_STREAM, 0);
    if(iSocketServer == -1){
        perror("socket");
        return -1;
    }

    // connect
    SocketServerAddr.sin_family = AF_INET;
    SocketServerAddr.sin_port = htons(Server_PORT);
    memset(SocketServerAddr.sin_zero, 0, 8);
    if(0 == inet_pton(AF_INET, argv[1], &SocketServerAddr.sin_addr)){
        printf("inValid IP_Addr! \n");
        return -1;
    }

    iRet = connect(iSocketServer, (struct sockaddr *)&SocketServerAddr, sizeof(struct sockaddr));
    if(iRet == -1){
        perror("connect");
        return -1;
    }

    //发送数据
    while(1){
        if(fgets(SendBuf, 999, stdin)){
            RecvLen = send(iSocketServer, SendBuf, strlen(SendBuf), 0);
            if(RecvLen <= 0){
                close(iSocketServer);
                return -1;
            }
        }
    }
    return 0;
}
```

### UDP

UDP与TCP的不同点在于UDP不需要connect阶段，而是创建 Socket -> 绑定地址 (bind) -> 接收/发送数据 -> 关闭连接

**server.c**

```c
#include <stdio.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <signal.h>

/*
*  创建 Socket -> 绑定地址 (bind) -> 接收/发送数据 -> 关闭连接
*/

#define Server_PORT 8888


int main(int argc, char **argv){
    int SocketServer;
    int SocketClient;
    int iRet;
    struct sockaddr_in SocketServerAddr;
    struct sockaddr_in SocketClientAddr;
    int iAddrLen;
	int iClientNum = -1;
    unsigned char RecVbuf[1000];
    int iRecvLen;

    //先建立一个socket
    SocketServer = socket(AF_INET, SOCK_DGRAM, 0);
    if(SocketServer == -1){
        perror("socket");
        return -1;
    }
    //绑定地址
    SocketServerAddr.sin_family = AF_INET;
    SocketServerAddr.sin_addr.s_addr = htonl(INADDR_ANY);
    SocketServerAddr.sin_port = htons(Server_PORT);
    memset(SocketServerAddr.sin_zero, 0, 8);

    iRet = bind(SocketServer, (const struct sockaddr *)&SocketServerAddr, sizeof(struct sockaddr));
    if(iRet == -1){
        perror("bind");
        return -1;
    }


    while(1){
        iAddrLen  = sizeof(struct sockaddr);
        iRecvLen = recvfrom (SocketServer, RecVbuf, 999, 0, (struct sockaddr *)&SocketClientAddr, &iAddrLen);
        if(iRecvLen > 0){
            RecVbuf[iRecvLen] = '\0';
            printf("Get MSG From %s: %s \n", inet_ntoa(SocketClientAddr.sin_addr), RecVbuf);
        }
    }
	close(SocketServer);
	return 0;
}
```

**client.c**

```c
#include <stdio.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

#define Server_PORT 8888

int main(int argc, char **argv)
{
    int iSocketServer;
    int iRet;
    struct sockaddr_in SocketServerAddr;
    unsigned char SendBuf[1000];
    int RecvLen;
    if (argc != 2)
    {
        printf("Usage: %s <ipaddr> \n", argv[0]);
        return -1;
    }

    // 创建Socket
    iSocketServer = socket(AF_INET, SOCK_DGRAM, 0);
    if (iSocketServer == -1)
    {
        perror("socket");
        return -1;
    }

    // connect
    SocketServerAddr.sin_family = AF_INET;
    SocketServerAddr.sin_port = htons(Server_PORT);
    memset(SocketServerAddr.sin_zero, 0, 8);
    if (0 == inet_pton(AF_INET, argv[1], &SocketServerAddr.sin_addr))
    {
        printf("inValid IP_Addr! \n");
        return -1;
    }
#ifdef Connect
    iRet = connect(iSocketServer, (struct sockaddr *)&SocketServerAddr, sizeof(struct sockaddr));
    if (iRet == -1)
    {
        perror("connect");
        return -1;
    }
    printf("UDP connect! \n");
#endif

    // 发送数据
    while (1)
    {
        if (fgets(SendBuf, 999, stdin))
        {
#ifdef Connect
            RecvLen = send(iSocketServer, SendBuf, strlen(SendBuf), 0);

#else
            RecvLen = sendto(iSocketServer, SendBuf, strlen(SendBuf), 0, (struct sockaddr *)&SocketServerAddr, sizeof(struct sockaddr));
#endif
            if (RecvLen <= 0)
            {
                close(iSocketServer);
                return -1;
            }
        }
    }
    return 0;
}
```

client 有两种用法，分别是使用 `connect` 和不使用 `connect` 函数。
