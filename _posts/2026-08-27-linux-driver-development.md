---
layout: post
title: Linux 驱动开发学习笔记
date: 2026-08-27 17:26:08 +0800
description: 系统整理 Linux 字符设备、I/O 模型、中断、设备树、platform 总线、configfs、设备模型与 kobject 等驱动开发基础。
tags: [Linux, 驱动开发, 内核, 设备树]
categories: [Linux学习]
published: true
toc:
  sidebar: left
---

## 驱动分类

1. 字符设备：必须以串行顺序进行访问的设备，如鼠标。
2. 网络设备：面向数据包的接收和发送。
3. 块设备：可以按任意顺序访问，如硬盘。
   Linux规定每一个字符设备或者块设备都必须有一个专属的设备号，一个设备号由主设备号和次设备号组成，主设备号表示某一类驱动，如鼠标键盘都可以归为USB驱动，而次设备号表示这个驱动下的各个设备。

## 编写一个驱动

1. 引入头文件\*
2. 编写驱动加载函数\*
3. 编写驱动卸载函数\*
4. 许可证声明\*
5. 模块参数
6. 作者和版本信息

```c
#include <linux/module.h>
#include <linux/init.h>

static int helloworld_init(void){
    printk("helloworld init!\n");
    return 0;
}

static void helloworld_exit(void){
    printk("helloworld exit!\n");
}


module_init(helloworld_init);
module_exit(helloworld_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("1oong7"); //以下为可选项
MODULE_VERSION("V0.1");
```

## 编写Makefile文件

```makefile
obj-m+=helloworld.o
KDIR:=/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel
PWD?=$(shell pwd)
all:
    make -C $(KDIR) M=$(PWD) modules
clean:
    $(MAKE) -C $(KDIR) M=$(PWD) clean
    rm -f *.ko *.o *.mod.o *.mod.c *.symvers *.order
```

编译指令：`make ARCH=arm64`
清理指令：`make clean`

## 加载模块命令

模块加载命令：`insmod helloworld.ko` 或者 `modprobe helloworld.ko` modprobe 可以加载依赖
模块卸载命令：`rmmod helloworld.ko`
查看模块信息：`lsmod` 或者`modinfo helloworld.ko`

## menuconfig图形化配置

命令： `make menuconfig`
config文件可分为三种：`defconfig`、`.config`、`Kconfig`
`defconfig` 是厂商或架构开发者提供的**默认配置文件**，作为配置的基准起点；
`Kconfig` 定义了内核配置系统的**菜单结构、选项依赖关系和帮助信息**，被 `menuconfig` 等工具读取生成图形界面；
`.config` 是内核源码根目录下的**实际配置文件**，存储了所有编译选项的最终值，当使用 `make menuconfig` 修改配置并保存后，会更新到 `.config` 文件中，内核编译时读取此文件决定编译哪些代码

## 驱动文件传参

可以传递的有普通参数、数组和字符串，分别使用`module_param` `module_param_array` `module_param_string`函数
`MODULE_PARM_DESC`为传递参数描述函数

```c
module_param(param, int, S_IRUGO);
MODULE_PARM_DESC(param, "e.g.: param = 10");

module_param_array(array, int, &array_size, S_IRUGO);
MODULE_PARM_DESC(array, "e.g.: array = 1,2,3,4...");

module_param_string(str,str1,sizeof(str1),S_IRUGO);
MODULE_PARM_DESC(array, "e.g.: str = hello");
```

## 内核模块符号导出

**解决B模块依赖于A模块的问题**：当A模块导出符号（使用 `EXPORT_SYMBOL`）后，编译B模块时需要知道这些符号的信息。需要先编译A模块生成 `Module.symvers` 文件，然后将该文件**复制或追加**到B模块的编译目录中，这样B模块在编译时就能解析A模块导出的符号。

## 添加系统调用到内核

1. 在内核源码中添加自己的服务，需要编译进内核
2. 添加系统调用号
3. 编译并烧写到开发板

## 字符设备基础

### 申请字符设备号

Linux使用一个`dev_t`的数据类型表示设备号，定义在`include/linux/types.h`里，为`unsigned int`类型，其中高12位表示主设备号，低20位表示次设备号。
相关宏定义

```c
#define MINORBITS 20 /*次设备号位数*/
#define MINORMASK ((1U << MINORBITS) - 1)  /*次设备号掩码*/
#define MAJOR(dev) ((unsigned int) ((dev) >> MINORBITS)) /*dev 右移 20 位得到主设备号*/
#define MINOR(dev) ((unsigned int) ((dev) & MINORMASK))  /*与次设备掩码与， 得到次设备号*/
#define MKDEV(ma,mi) (((ma) << MINORBITS) \| (mi)) /*MKDEV 宏将主设备号（ma） 左移 20 位， 然后与次设备号（mi） 按位或， 得到设备号*/
```

在 Linux 驱动中可以使用以下两种方法进行设备号的申请。

1. 通过 `register_chrdev_region(dev_t from, unsigned count, const char *name)`函数进行**静态**申请设备号。
   1. from: 自定义的 `dev_t` 类型设备号
   2. count: 申请设备的数量
   3. name: 申请的设备名称
      函数返回值： 申请成功返回 0， 申请失败返回负数
2. 通 过 `alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,const char *name)`函数进行**动态**申请设备号。
   函数作用：动态申请设备号， 内核会自动分配一个未使用的设备号， 相较于静态申请设备号， 动态申请会避免注册设备号相同引发冲突的问题。1. dev \*: 会将申请完成的设备号保存在 dev 变量中 2. baseminor: 次设备号可申请的最小值 3. count: 申请设备的数量 4. name: 申请的设备名称
   函数返回值： 申请成功返回 0， 申请失败返回负数

```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/moduleparam.h>
#include <linux/fs.h>
#include <linux/kdev_t.h>

static int major = 0;
static int minor = 0;

module_param(major, int, S_IRUGO);
module_param(minor, int, S_IRUGO);

dev_t dev_num;

static int dev_init(void){
    int ret;
    if(major){
        printk("major is %d \n",major);
        printk("minor is %d \n",minor);
        dev_num = MKDEV(major, minor);
        ret = register_chrdev_region(dev_num, 1, "chrdev_name");
        if(ret < 0){
            printk("register_chrdev_region error! \n");
            return -1;
        }
        else
        {
            printk("register_chrdev_region ok! \n");
        }
    } else {
        ret = alloc_chrdev_region(&dev_num, 0, 1, "alloc_name");
        if(ret < 0){
            printk("alloc_chrdev_region error! \n");
            return -1;
        }
        else
        {
            printk("alloc_chrdev_region ok! \n");
            major = MAJOR(dev_num);
            minor = MINOR(dev_num);
            printk("major is %d \n",major);
            printk("minor is %d \n",minor);
        }
    }
    return 0;
}

static void dev_exit(void){
    unregister_chrdev_region(dev_num, 1);
    printk("chrdev exit!\n");
}

module_init(dev_init);
module_exit(dev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("1oong7");
MODULE_VERSION("V0.1");

```

### 注册字符类设备

注册字符类设备节点需要在完成申请字符设备号之后。
首先需要定义一个`cdev`和`file_operatios`的结构体，然后对结构体进行赋值，然后调用`cdev_init`函数进行初始化，最后调用`cdev_add`函数添加到内核。

1. **定义与赋值**：定义 `struct cdev` 和 `struct file_operations`，并将你的驱动函数（如 `.open`、`.read`）赋值给 `fops`。
2. **初始化**：调用 `cdev_init(&cdev, &fops)` 将 `cdev` 与 `fops` 绑定。
3. **添加**：调用 `cdev_add(&cdev, dev_t, count)` 将设备告知内核。

卸载设备时，需要先删除`cdev`，然后再释放设备号。

### file_operations结构体

`struct file_operations`结构体存在`include/linux/fs.h`里，常用的有以下几个：

```c
struct file_operations {
    struct module *owner;
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
}
```

对应使用为：

```c
static struct file_operations cdev_chrdev_ops={
    .owner = THIS_MODULE,
    .open  = cdev_chrdev_open,
    .read  = cdev_chrdev_read,
    .write  = cdev_chrdev_write,
    .release  = cdev_chrdev_release
};
```

### 设备节点

在 Linux 操作系统中一切皆文件， 对于用来进行设备访问的文件称之为设备节点， 设备节点被创建在/dev 目录下， 根据设备节点的创建方式不同， 分为了手动创建设备节点和自动创建设备节点。

1. 手动创建设备节点命令`mknod NAME TYPE MAJOR MINOR`,
   其中，NAME表示节点名称；TYPE里，`b`表示块设备，`c`表示字符设备；MAJOR和MINOR分别表示主设备号和次设备号。
   示例：`mknod /dev/chrdev c 236 0`
2. 自动创建设备节点----`udev`机制
   自动创建设备节点是利用 udev 机制来实现的。 udev 通过检测系统中硬件设备状态， 可以根据系统中硬件设备状态来创建或者删除设备文件。 自动创建设备节点需要在驱动中首先使用`class_create()`函数创建一个类， 创建的类位于于/sys/class/ 目录下， 之后使用 `device_create()`函数在这个类下创建相应的设备， 在加载驱动模块时， 用户空间中的 udev 会自动响应根据并/sys/class/ 下的信息创建设备节点。

   ```c
   class_create(owner, name);
   struct device *device_create(struct class *class, struct device *parent,

                dev_t devt, void *drvdata, const char *fmt, ...);
   ```

常用函数：`copy_to_user`和`copy_from_user`，一个是从内核往用户传数据，另一个是从用户层获取数据。

### private_data私有数据

多个相同类型的设备经常会共用同一个驱动。 如果使用全局变量存储设备结构体则会导致不同进程间数据冲突。 这时使用文件私有数据可以防止不同设备间的数据互相干扰， 即在对每个设备进行操作时， 每个设备都申请自己的私有数据。 这样在操作设备时不会产生数据冲突。
使用时一般在open函数里将设备结构体指向 file 结构体中 private_data 成员， 此时设备结构体就变成了文件私有数据：`file->private_data = &dev1;`
然后通 file 结构体可以将私有数据一路从 open()函数带到 read(), write()函数。 接着就是可以在 read()、 write()函数中使用文件私有数据。
`struct device_test *test_dev=(struct device_test *)file->private_data;`

### container_of函数

container_of()函数可以通过结构体变量中某个成员的首地址进而获得整个结构体变量的首地址。
它的数学逻辑非常简单：

$$\text{结构体首地址} = \text{成员变量的实际地址} - \text{该成员在结构体中的偏移量(Offset)}$$

> 函数原型：`container_of(ptr ,type, member)`
> 函数作用：通过结构体变量中某个成员的首地址获取到整个结构体变量的首地址。
> 参数含义：`ptr `是结构体变量中某个成员的地址；`type` 是结构体的类型 ；`member` 是该结构体变量的具体名字。

假设有两个设备dev1和dev2，可以使用container_of()函数来确定是open哪个设备，示例代码如下：

```c
static     int cdev_chrdev_open(struct inode *inode, struct file *file){
    dev1.minor = 0;
    dev2.minor = 1;
    //inode->i_rdev 为该 inode 的设备号 使用 container_of 函数找到结构体变量 dev1 dev2 的地址 然后设置私有数据
    file->private_data = container_of(inode->i_cdev, struct device_test, cdev_chrdev);
    printk("this is cdev_chrdev_open\n");
    return 0;
}
```

### 点亮一个LED灯

先查看底板原理图，找到LED对应的引脚为`GPIO0_C0`，输出低电平时熄灭，输出高电平时点亮。
echo none > /sys/devices/platform/leds/leds/work/trigger
![RK3568 LED GPIO 配置]({{ '/assets/img/blog/linux-driver-development/rk3568-led-gpio.png' | relative_url }})
根据数据手册，`GPIOC_C0`在`PMU_GRF_GPIO0C_IOMU X_L` 的2:0位，根据基地址`0xFDC20000`和地址偏移`0x0010`计算实际地址为`0xFDC20010`。
fdc20010: 00000010
GPIO0 的基地址为 0xFDD60000 ，方向控制寄存器的偏移地址为0x000C，最终地址为0xFDD6000C.
输出控制寄存器的地址为0xFDD60004

## 并发、竞争与 I/O 模型详解

## 并发与竞争

### 原子操作

原子操作是最小执行单位，执行原子操作时不可被打断、抢占。

### 自旋锁

自旋锁是为了保护共享资源提出的一种锁机制。 当一个线程尝试获取自旋锁而发现该锁已被其他线程持有时， 它不会进入睡眠状态等待， 而是会持续循环地尝试获取锁， 直到成功为止。 这个过程称为“自旋” ， 因此得名“自旋锁” 。
以原地等待的方式解决资源冲突。

- **铁律一：自旋锁保护的临界区绝对不能休眠！** 一旦在持有自旋锁期间调用了可能引发休眠的函数（如 `copy_from_user`、`copy_to_user`、`kmalloc(…, GFP_KERNEL)`、`msleep()` 等），或者尝试去获取互斥锁（Mutex），系统将直接崩溃或死锁。
- **铁律二：持有锁的时间一定要短！** 自旋锁在等待时会百分之百占用 CPU。如果临界区包含复杂的循环、耗时的 I/O 操作，会导致其他 CPU 核心在原地空转，严重拉低系统吞吐量。
- **铁律三：必须考虑死锁递归！** 同一个线程绝对不能在未释放自旋锁的情况下，再次尝试获取同一把自旋锁。自旋锁是不可重入的，这样做会导致自己把自己卡死。

#### 自旋锁的死锁

自旋锁死锁是指多个任务或进程因互相等待彼此释放资源， 导致谁也无法继续往下执行的现象。举个例子，假设有进程 A 和进程 B 俩个进程，进程 A 对资源 1 持有自旋锁， 同时进程 A也想获取资源2。 而进程 B 对资源 2 持有自旋锁， 同时进程 B 也想获取资源 1。 在这种情况下进程 A 和进程 B 都在等待彼此释放资源， 从而造成了死锁。

又如进程 A 在拥有自旋锁期间发生了中断， CPU 转头去执行中断处理函数， 而此时的中断处理函数也需要获取同一把自旋锁。 这时由于锁已被 A 占用， 中断处理函数只能等待自旋，从而导致系统进入死锁状态。

所以， 在使用自旋锁的时候要格外注意死锁问题。 如在中断处理函数中也要获得自旋锁时，驱动程序则需要在拥有自旋锁时禁止中断， 在释放自旋锁时使能中断。 同时尽量短时间拥有自旋锁， 长时间拥有自旋锁可能会导致系统资源耗尽从而造成死锁。 最后也要避免某个获得自旋锁的函数调用其他同样试图获取这个锁的函数， 否则也会造成死锁。

### 信号量

### 互斥锁

- **铁律一：绝对不能在中断上下文中使用互斥锁！** 因为硬件中断服务程序（ISR）和软中断是不允许休眠的。如果尝试在中断里调用 `mutex_lock`，内核会直接崩溃（Kernel Panic）。
- **铁律二：不能在持有自旋锁（Spinlock）时去获取互斥锁！** 自旋锁内严禁休眠，而互斥锁可能引发休眠，两者水火不容。
- **铁律三：不要重复解锁，也不要在未加锁时解锁。** 互斥锁必须由**加锁的同一个进程**来负责解锁（谁拿的锁，谁负责还）。

## IO模型

阻塞IO、非阻塞IO、IO多路复用、信号驱动IO、异步IO
![Linux I/O 模型]({{ '/assets/img/blog/linux-driver-development/io-models.png' | relative_url }})

### 阻塞式IO 和 非阻塞式IO

通过等待队列来实现。
在 Linux 驱动开发中，**等待队列（Wait Queue）** 是实现“阻塞访问”的标准机制。当应用程序读取驱动数据而数据未就绪时，驱动会让进程进入休眠（阻塞）；当硬件产生中断、或者有数据写入导致数据就绪时，驱动再唤醒等待队列中的进程，使其继续执行。

实现等待队列阻塞访问的核心流程可以总结为：**定义与初始化、在 `read` 中条件不满足时休眠、在 `write` 或中断中数据就绪时唤醒。**

- **休眠等待（条件不满足时）：** `wait_event_interruptible(wq_head, condition)`
  - `wq_head`：等待队列头。
  - `condition`：等待条件。如果为 `false`（0），进程进入**可中断的休眠**；如果为 `true`（非0），进程直接通过，不休眠。
  - _注意：该宏能够被用户信号（如 `Ctrl+C`）打断，被打断时返回非 0 值。_
- **唤醒队列（数据就绪时）：** `wake_up_interruptible(&wq_head)`
  - 唤醒 `wq_head` 队列中所有处于可中断休眠（`TASK_INTERRUPTIBLE`）状态的进程。

非阻塞IO在阻塞IO的基础上多了一个`O_NONBLOCK`关键词。

```c
static     ssize_t cdev_chrdev_read(struct file *file, char __user *buf, size_t size, loff_t *off){

    struct device_test *dev_test = (struct device_test* )file->private_data;
    size_t len;
    if(file->f_flags & O_NONBLOCK){   //判断是否有O_NONBLCOK关键词 有则进入非阻塞IO模式，不停查询返回EAGAIN
        if(!dev_test->data_ready){
            return -EAGAIN;
        }
    }
    else {                            //没有关键词，则进入正常阻塞模式，在此休眠，等待write函数里唤醒休眠
        if(wait_event_interruptible(dev_test->read_wq, dev_test->data_ready)){
            return -ERESTARTSYS;
        }
    }

    len = strlen(dev_test->kbuf);
    if(copy_to_user(buf, dev_test->kbuf,len) != 0){
        printk("copy_to_user error \n");
        return -1;
    }

    dev_test->data_ready = 0;
    return len;
}
static    ssize_t cdev_chrdev_write(struct file *file, const char __user *buf, size_t size, loff_t *off){
    struct device_test *dev_test = (struct device_test* )file->private_data;
    size_t copy_size = (size > 31) ? 31 : size;
    if(copy_from_user(dev_test->kbuf, buf, copy_size) != 0){
        printk("copy_from_user error \n");
        return -EFAULT;
    };

    dev_test->kbuf[copy_size] = '\0'; // 确保字符串安全截断

    dev_test->data_ready = 1;         // 将标志位改为1，然后唤醒队列里所有处于可中断休眠状态的进程
    wake_up_interruptible(&dev_test->read_wq);

    return size;
}
```

### IO多路复用

常用的有`poll` `select` `epoll`三种,区别是前两种是遍历的，`epoll` 效率更高。
`poll`使用流程：先在驱动实现 file_operations 结构体中的 poll()函数，然后在应用中初始化`pollfd`结构体，调用poll函数监听。

```c
//驱动
__poll_t (*poll) (struct file *file, struct poll_table_struct *p);
void poll_wait(struct file *filp,wait_queue_head_t *queue,poll_table *wait);

static __poll_t cdev_chrdev_poll(struct file *file, struct poll_table_struct *p){
    struct device_test *dev_test = (struct device_test* )file->private_data;
    __poll_t mask = 0;
    poll_wait(file, &dev_test->read_wq, p);
    if(dev_test->data_ready == 1){
        mask |= POLLIN;
    }
    return mask;
}

//结构体
struct pollfd {
int fd; //被监视的文件描述符
short events; //等待的事件
short revents; //实际发生的事件
};

//初始化
fds[0].fd = fd1;
fds[0].events = POLLIN;
fds[1].fd = fd2;
fds[1].events = POLLIN;
while(1){
    ret = poll(fds, 2, 5000); // Wait for 5 seconds
    ···
}
```

### 信号驱动IO

信号驱动 IO 指的是进程会**预先告知内核**， 当某个描述符发生事件时， 内核要向该进程发送 SIGIO 信号进行通知， 进程可以在**信号处理函数**中对该事件进行处理。
要使用信号驱动 IO， 需要应用程序和驱动程序配合， 应用程序使用信号驱动 IO 的步骤有三步：

> 步骤 1： 注册信号处理函数， 应用程序中使用 signal()函数来注册 SIGIO 信号的信号处理函数。
> 步骤 2： 使用 fcntl()函数设置能够接收这个信号的进程。
> 步骤 3： 使用 fcntl()函数的 F_SETFL 参数打开 FASYNC 标志开启信号驱动 IO。

```c
//注册信号处理函数
    signal(SIGIO, read_func);
    //把进程id告诉驱动
    fcntl(fd1, F_SETOWN, getpid());
    //使能FASYNC
    flags = fcntl(fd1,F_GETFL);
    fcntl(fd1, F_SETFL, flags | FASYNC);
```

驱动程序使用信号驱动 IO 的步骤也包括三步:

> _步骤 1:_
> 应用程序使用信号驱动 IO 时， 会触发驱动中的 fasync()函数。 因此要在 file_operations 结构体中实现 fasync()函数， fasync()函数原型为如下：
> `int (*fasync) (int fd,struct file *filp,int on)`

步骤 2:

> 在 驱 动 的 fasync() 函 数 中 调 用 fasync_helper() 函 数 来 初 始 化 fasync_struct 结 构 体 ， fasync_helper()函数原型如下：
> `int fasync_helper(int fd,struct file *filp,int on,struct fasync_struct **fapp)`
>
> 步骤 3：
> 当设备准备好（即可以访问） 时， 驱动程序需要调用 kill_fasync()函数通知应用程序， 此时应用程序的 SIGIO 信号处理函数就会被执行。 kill_fasync()负责发送指定的信号， 函数原型如下：
> ` void kill_fasync(struct fasync_struct **fp,int sig,int band)`
> 函数参数：
> fp: 要操作的 fasync_struct
> sig: 发送的信号
> band: 事件类型， 可读的时候设置成 POLLIN ， 可写的时候设置成 POLLOUT

## 定时器

## 内核打印

`dmesg | grep hello`
-C， --clear 清除内核环形缓冲区
-c， —-read-clear 读取并清除所有消息
-T， --显示时间戳
使用`cat /proc/sys/kernel/printk`命令查看当前默认打印等级.
![printk 默认日志级别]({{ '/assets/img/blog/linux-driver-development/printk-default-levels.png' | relative_url }})
![printk 日志级别控制]({{ '/assets/img/blog/linux-driver-development/printk-level-controls.png' | relative_url }})
使用方法：

1. printk(打印等级 "打印信息")
2. echo 4 4 1 7 > /proc/sys/kernel/printk

## llseek定位

## ioctl驱动传参

## 优化驱动稳定性和效率

1. 检测ioctl命令
2. 使用access_ok判断传递地址是否合理
3. 使用likely和unlikely进行分支预测优化

## 驱动调试方法实验

1. dump_stack 函数
   作用：打印内核调用堆栈， 并打印函数的调用关系。
2. WARN_ON (condition)函数
   在括号中的条件成立时， 内核会抛出栈回溯， 打印函数的调用关系。因此通常用于内核抛出一个警告， 暗示某种不太合理的事情发生了。
3. BUG_ON函数
   内核中有许多地方调用了 BUG_ON()的语句， 它非常像一个内核运行时的断言， 意味着本来不该执行到 BUG_ON()这条语句但是确执行了， 一旦 BUG_ON()函数执行内核就会立刻抛出**oops**。 BUG_ON()函数的参数 condition 用来判断条件是否成立， 当 condition 为非 0（真） 时，立即触发 BUG； 当 condition 为 0（假） 时， 继续执行程序。 例如 BUG_ON (1)表示条件判断成功， 立即触发 BUG。
4. panic(fmt…)函数
   panic (fmt…)函数可以将函数的调用关系以及寄存器值就都打印出来， 并且会造成系统死机

## 中断

![GIC 架构]({{ '/assets/img/blog/linux-driver-development/gic-architecture.png' | relative_url }})
ARM 多核处理器里最常用的中断控制器是 **GIC（Generic Interrupt Controller）** ， 用于管理和控制中断。 它接收来自各种**中断源**的中断请求， 并根据一定的设置策略， 将中断请求分发给对应的 CPU 进行处理。
在 RK3568 上使用的 GIC 版本为GIC-V3。

GIC-V3 支持四种类型的中断， 分别是 SGI、 PPI、 SPI 和 LPI， 每种中断类型的介绍如下：

- SGI(Software Generated Interrupt): 软中断， 主要用于核间通信， 通过写 SGI 寄存器产生。
- PPI(Private Peripheral Interrupt):私有外设中断， 为某个核的私有中断。
- SPI(Shared Peripheral Interrupt):共享外设中断， 外设中断可以发送到任何一个连接的 core。
- LPI(Locality-specific Peripheral Interrupt):特定区域外设中断。 只在 GICv3 和 GICv4 上支持
  ![GIC 中断类型]({{ '/assets/img/blog/linux-driver-development/gic-interrupt-types.png' | relative_url }})

### 申请一个GPIO中断

在init里使用`requset_irq()`函数申请中断，其中包含对GPIO的irq计算，中断回调函数的使用，中断标志位选择等。
在exit里使用`free_irq()`卸载中断。

```c
int irq;
irqreturn_t interrupt_func(int irq, void *arg){
    printk("this is interrupt_func\n");
    return IRQ_RETVAL(IRQ_HANDLED);
}
static int interrupt_irq_init(void){
    int ret;
    irq = gpio_to_irq(21);
    printk("irq is %d", irq);
    ret = request_irq(irq, interrupt_func, IRQF_TRIGGER_FALLING, "LVDS_INT",NULL);
    if(ret < 0){
        printk("request_irq error\n");
        return -1;
    }
    dump_stack();
    return 0;
}

static void interrupt_irq_exit(void){
    free_irq(irq, NULL);
    printk("interrupt_irq_exit!\n");
}
```

### tasklet中断下文

在 Linux 内核中， tasklet 是一种特殊的软中断机制， 被广泛用于处理中断下文相关的任务。tasklet 绑定的函数在同一时间只能在一个 CPU 上运行， 因此不会出现并发冲突。 tasklet 绑定的函数中不能调用可能导致休眠的函数， 否则可能引起内核异常。

```c
struct tasklet_struct {
    struct tasklet_struct *next; //指向下一个 tasklet 的指针， 用于形成链表结构
    unsigned long state; //表示 tasklet 的当前状态
    atomic_t count; //用于引用计数， 用于确保 tasklet 在多个地方调度或取消调度时的正确处理。
    void (*func)(unsigned long); //指向 tasklet 绑定的函数的指针， 该函数将在 tasklet 执行时被调用。
    unsigned long data; //传递给 tasklet 绑定函数的参数
};

typedef struct tasklet_struct tasklet_t; //定义了 tasklet_t 类型作为 struct tasklet_struct 的别名|
```

使用方法基本就是调用`DECLARE_TASKLET` 宏来静态初始化 tasklet，或者使用`tasklet_init()`动态初始化tasklet，然后`tasklet_enable()`使能、`tasklet_disable()`失能tasklet，在需要调度tasklet时，使用`tasklet_schedule()`函数来调度一个已经初始化的 tasklet。下边分别给出他们的函数原型，其中的参数显而易见。

```c
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

void tasklet_init(struct tasklet_struct *t, void (*func)(unsigned long), unsigned long data);

static inline void tasklet_disable(struct tasklet_struct *t);
static inline void tasklet_enable(struct tasklet_struct *t);

static inline void tasklet_schedule(struct tasklet_struct *t);
```

```c
int irq;
struct tasklet_struct mytasklet;

//中断下文
void mytasklet_func(unsigned long data){
    printk("data is %ld", data);
}

//中断上文
irqreturn_t interrupt_func(int irq, void *arg){
    printk("this is interrupt_func\n");
    tasklet_schedule(&mytasklet);
    return IRQ_RETVAL(IRQ_HANDLED);
}

static int interrupt_irq_init(void){
    int ret;
    irq = gpio_to_irq(32); //将GPIO转为中断号
    printk("irq is %d", irq);

    gpio_request(32, "my_test_irq");
    gpio_direction_input(32);

    ret = request_irq(irq, interrupt_func, IRQF_TRIGGER_FALLING, "GPIO1_A0",NULL); //请求中断
    if(ret < 0){
        printk("request_irq error\n");
        return -1;
    }

    tasklet_init(&mytasklet, mytasklet_func, 1); //初始化tasklet，并传入参数 1
    return 0;
}

static void interrupt_irq_exit(void){

    free_irq(irq, NULL);
    tasklet_enable(&mytasklet);
    tasklet_kill(&mytasklet);
    printk("interrupt_irq_exit!\n");
}
```

### 软中断

```c
/* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
   frequency threaded job scheduling. For almost all the purposes
   tasklets are more than enough. F.e. all serial device BHs et
   al. should be converted to tasklets, not to softirqs.
 */
enum
{
    HI_SOFTIRQ=0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    IRQ_POLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
                numbering. Sigh! */
    RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

    NR_SOFTIRQS
};
```

Linux 内核的开发者并不希望我们这样去做，如果要用软中断， 建议使用 tasklet。tasklet是一种特殊的软中断。

### 工作队列

工作队列也是实现函数中断下文的机制，可以执行更耗时的任务，**可以使用休眠**。
工作队列可分为**共享工作队列**和**自定义工作队列**。
共享队列是由内核管理的全局工作队列， 用于处理内核中一些系统级任务。 共享工作队列是内核中一个默认工作队列， 可以由多个驱动程序共享使用。
自定义工作队列是由内核或驱动程序创建的特定工作队列， 用于处理特定的任务。 自定义工作队列通常与特定的内核模块或驱动程序相关联， 用于执行该模块或驱动程序相关的任务。

```c
// 1. 声明：一个是公司，一个是包裹
struct workqueue_struct *workqueue_share;
struct work_struct my_work;

// 2. 驱动初始化时：
workqueue_share = create_workqueue("share_queue"); // 创建公司
INIT_WORK(&my_work, my_callback_func);         // 把任务和回调函数绑定

// 3. 发生中断或需要异步执行时（寄快递）：
queue_work(workqueue_share, &my_work);          // 把包裹投递到这个公司里

// 4. 驱动退出时：
destroy_workqueue(workqueue_share);                // 关停公司
```

#### 共享工作队列

```c
int irq;
struct work_struct work_queue;

void work_func(struct work_struct *work){
    msleep(1000);
    printk("this is work_func \n");

}

irqreturn_t interrupt_func(int irq, void *arg){
    printk("this is interrupt_func\n");
    schedule_work(&work_queue);
    return IRQ_RETVAL(IRQ_HANDLED);
}

static int interrupt_irq_init(void){
    int ret;
    irq = gpio_to_irq(32);
    printk("irq is %d", irq);

    gpio_request(32, "my_test_irq");
    gpio_direction_input(32);

    ret = request_irq(irq, interrupt_func, IRQF_TRIGGER_FALLING, "GPIO1_A0",NULL);
    if(ret < 0){
        printk("request_irq error\n");
        return -1;
    }

    INIT_WORK(&work_queue, work_func);    //init
    return 0;
}
```

#### 自定义工作队列

```c
int irq;
struct workqueue_struct *workqueue_share;
struct work_struct work_queue;

void work_func(struct work_struct *work){
    msleep(1000);
    printk("this is work_func !!!!\n");

}

irqreturn_t interrupt_func(int irq, void *arg){
    printk("this is interrupt_func\n");
    queue_work(workqueue_share,&work_queue);   //调度自定义工作队列
    return IRQ_RETVAL(IRQ_HANDLED);
}

static int interrupt_irq_init(void){
    int ret;
    irq = gpio_to_irq(32);
    printk("irq is %d", irq);

    gpio_request(32, "my_test_irq");
    gpio_direction_input(32);

    ret = request_irq(irq, interrupt_func, IRQF_TRIGGER_FALLING, "GPIO1_A0",NULL);
    if(ret < 0){
        printk("request_irq error\n");
        return -1;
    }

    workqueue_share = create_workqueue("workqueue_test");  //创建自定义队列
    INIT_WORK(&work_queue, work_func); //init
    return 0;
}

static void interrupt_irq_exit(void){

    free_irq(irq, NULL);
    cancel_work_sync(&work_queue);
    flush_workqueue(workqueue_share); // 刷新工作队列
    destroy_workqueue(workqueue_share); // 销毁工作队列
    printk("interrupt_irq_exit!\n");
}
```

#### 工作队列传参

工作队列里并没有带参数传递的函数，但是`work_func(struct work_struct *work)`传入了结构体，因此可以使用结构体传参。
首先定义一个大的结构体，然后在`work_func()`里使用`container_of`获取该结构体的首地址，即可使用该结构体的参数。

```c
struct my_deviece_data{
    struct work_struct irq_work;
    int param;
    int data_1;
};

void work_func(struct work_struct *work){
    struct my_deviece_data *pdata;
    pdata = container_of(work,struct my_deviece_data, irq_work);

    ···
}
```

### 延迟工作

相当于调度之后，延迟一定时间后再执行，具体可参照按键消抖。
使用方法也是**先静态或动态初始化**，然后绑定中断，在**中断上文中调度**，不同于工作队列的是**需要传入一个延迟时间**，其他类似。

### 并发管理工作队列

[[Linux-CMWQ]]

### 中断线程化

中断线程化是 Linux 内核中优化中断处理的一种机制。 传统的中断处理分成了顶半部和底半部， 顶半部用来处理比较**紧急但是又不是很耗时**的事情， 如设置标志位， 底半部用来处理**耗时的事情**。
在中断线程化机制中， 中断处理程序不再执行具体任务， 而是只负责**唤醒一个内核线程**来完成主要的中断处理逻辑。 这样可以显著缩短中断响应时间， 使被中断的实时进程能够**更快地恢复运行**。 因此中断线程化也被认为是一种实现底半部的方式。
[[Linux-中断线程化与并发管理工作队列对比]]
**中断线程化的使用非常简单， 只需要在申请中断时调用 request_threaded_irq()函数**。
`int request_threaded_irq(|unsigned|int irq, irq_handler_t handler,irq_handler_t thread_fn, unsigned long irqflags,const char *devname, void *dev_id);`

> 函数作用：注册中断处理程序并实现中断线程化。
> 函数参数：
> irq： 要请求的中断号。handler： 中断上半部分函数。
> thread_fn： 中断线程， 如果此处设置为 NULL， 则表示没有使用中断线程化。irqflags： 中断标志位。
> devname： 中断的名称。dev_id： 共享中断时使用。
> 返回值：函数返回一个整数值， 表示中断请求的结果。 如果中断请求成功， 返回值为 0， 否则返回复数错误代码。

与`requset_irq()`相比，只多了一个`thread_fn`中断线程参数。同时，在handler里需要返回`IRQ_WAKE_THREAD`，表示唤醒中断线程，同时在中断线程`thread_fn`返回`IRQ_RETVAL(IRQ_HANDLED);`

```c
int irq;

irqreturn_t thread_func(int irq, void *arg){
    ···
    return IRQ_RETVAL(IRQ_HANDLED);
}
irqreturn_t interrupt_func(int irq, void *arg){
    ···
    return IRQ_WAKE_THREAD;
}
static int interrupt_irq_init(void){
    ···
    ret = request_threaded_irq(irq, interrupt_func,thread_func, IRQF_TRIGGER_FALLING, "GPIO1_A0",NULL);
    ···
}

static void interrupt_irq_exit(void){
    ···
    free_irq(irq, NULL);
    ···
}
```

## platform平台模型

platform平台模型将硬件分成了两部分，device负责硬件相关部分，driver负责驱动控制部分，两者之间通过name匹配。

```c
/*
const char *name ： platform 设 备 的 名 称 ， 如 果 platform_device 中 的 name 成 员 与platform_driver 结构体 device_driver 成员中的 name 名称相同则会与对应的 platform 驱动匹配成功， 从执行 platform 驱动中的 probe 函数。
int id： platform 设备的 ID， 可以用于区分同一种 platform 设备的不同实例。 这个参数是可选的， 如果不需要使用 ID 进行区分， 可以将其设置为-1，
struct device dev： 表示 platform 设备对应的 struct device 结构体。
u32 num_resources： platform 设备需要使用的硬件资源数量。 如寄存器地址、 中断号等资源的总数。
struct resource *resource： 指向 struct resource 结构体的指针， struct resource 结构体的用来描述平台设备的硬件资源。 如描述地址寄存器地址， 中断号等
*/
#include <linux/platform_device.h>

struct platform_device {
    const char    *name; //name
    int        id;
    bool        id_auto; //自动设置id
    struct device    dev;
    u32        num_resources; //存储的资源的个数
    struct resource    *resource; //存储的资源

    const struct platform_device_id    *id_entry;
    char *driver_override; /* Driver name to force a match */

    /* MFD cell pointer */
    struct mfd_cell *mfd_cell;

    /* arch specific additions */
    struct pdev_archdata    archdata;
};
/*
resource_size_t start： 资源的起始地址。 如寄存器的起始地址。
resource_size_t end： 资源的结束地址。 如寄存器的结束地址。
const char *name： 资源的名称。 它是一个字符串， 用于标识和描述资源。
unsigned long flags： 资源的标志位。 有以下常见的几个标志位。
    IORESOURCE_IO： 表示资源是 I/O 端口资源。
    IORESOURCE_MEM： 表示资源是内存资源。
    IORESOURCE_REG： 表示资源是寄存器地址。
    IORESOURCE_IRQ： 表示资源是中断资源。
    IORESOURCE_DMA： 表示资源是 DMA（直接内存访问） 资源
*/
struct resource {
    resource_size_t start; //起始信息
    resource_size_t end;  //终止信息
    const char *name; //名字
    unsigned long flags; //存储的资源的类型
    unsigned long desc;
    struct resource *parent, *sibling, *child;

    ANDROID_KABI_RESERVE(1);
    ANDROID_KABI_RESERVE(2);
    ANDROID_KABI_RESERVE(3);
    ANDROID_KABI_RESERVE(4);
};
```

注册使用`int platform_device_register(struct platform_device *pdev);`，卸载使用`void platform_device_unregister(struct platform_device *pdev):`函数。

```c
/*
probe： 探测函数指针。 当 platform 驱动与 platform 设备匹配成功后会执行 probe 函数指针指向的函数。

remove： 移除函数指针。 当平台设备从系统中移除时， 该函数指针指向的函数将被调用以执行清理和释放资源的操作。

shutdown： 关闭函数指针。 当系统关闭时， 该函数指针指向的函数将被调用以执行与平台设备相关的关闭操作。

suspend： 挂起函数指针。 当系统进入挂起状态时， 该函数指针指向的函数将被调用以执行与平台设备相关的挂起操作。

resume： 恢复函数指针。 当系统从挂起状态恢复时， 该函数指针指向的函数将被调用以执行与平台设备相关的恢复操作。

struct device_driver driver： 表示 platform 驱动对应的 struct device_driver 结构体， struct device_driver 结构体中包括驱动程序的名称、 总线类型、 模块拥有者、 属性组数组指针等信息，该结构体的 name 参数需要与 platform_device 的.name 参数相同才能匹配成功， 从而执行 probe函数。

id_table： 指向 struct platform_device_id 结构体数组的指针， 当 platform 设备与 platform 驱动匹配时候， 如果存在 id_table 成员， 则优先使用 id_table 进行匹配， 如果 id_table 为 NULL，则使用 driver.name 成员与 platform_device.name 进行匹配。

prevent_deferred_probe： 一个布尔值， 用于确定是否阻止延迟探测。 如果设置为 true，则延迟探测将被禁用。
*/

#include <linux/platform_device.h>
#include <linux/mod_devicetable.h>
struct platform_driver {
    int (*probe)(struct platform_device *);
    int (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
};
```

注册使用`int platform_driver_register(struct platform_driver *drv);`，卸载使用`void platform_driver_unregister(struct platform_driver *drv):`函数。

### probe函数

在 platform 设备和 platform 驱动匹配成功之后， 如果需要使用在 platform 设备中描述的硬件资源， 可以通过 API 函数在 platform_driver 驱动的 probe 函数中获取在 platform 设备中描述的硬件资源。
使用 `platform_get_resource()` 获取硬件资源。`platform_get_resource()`函数用于从平台设备的资源数组中获取指定类型和索引的资源。 在平台设备的资源数组中， 每个元素都是一个 struct resource 结构体， 每个 struct resource 结构体描述了一个资源的信息， 如起始地址、 结束地址、 中断号等。

```c
struct resource *myresources;

int mydriver_probe(struct platform_device *dev){
    myresources = platform_get_resource(dev, IORESOURCE_MEM,0);
    printk("IORESOURCE_MEM is %0llx\n", myresources->end);
}
```

## 设备树

设备树是为了解决在内核里的**平台总线device部分**越来越臃肿，并且每次修改完之后都要**重新编译内核**而推出的方案。设备树文件不会被直接编译进内核， 而是以外部文件形式提供。

### 设备树基础知识

学习设备树（Device Tree） 时， 通常会涉及到 DTS、 DTSI、 DTB 和 DTC 等术语： 下面对每个术语进行介绍。
DTS（Device Tree Source） ： DTS 是设备树的**源文件**， 采用一种类似于文本的语法来描述硬件设备的结构、 属性和连接关系。 DTS 文件以.dts 为扩展名， 通常由开发人员编写。
DTSI（Device Tree Source Include） ： DTSI 文件是设备树**源文件的包含文件**。 它扩展了 DTS文件的功能， 用于定义可重用的设备树片段。 DTSI 文件以.dtsi 为扩展名， 可以在多个 DTS 文件中包含和共享。
DTB（Device Tree Blob） ： DTB 是设备树的二进制表示形式。 DTB 文件是通过将 DTS 或 DTSI文件**编译而成**的二进制文件， 以.dtb 为扩展名。
DTC（Device Tree Compiler） ： DTC 是设备树的**编译器**。 它是一个命令行工具， 用于将 DTS和 DTSI 文件编译成 DTB 文件。
![设备树编译流程]({{ '/assets/img/blog/linux-driver-development/device-tree-build-flow.png' | relative_url }})
ARM64 体系结构下的设备树源文件通常存放在内核源码` kernel/arch/arm64/boot/dts/`目录及其子目录中。 该目录也是设备树源文件的根目录， 并包含了针对不同 ARM64 平台和设备的子目录。

DTC编译器路径为`/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel/scripts/dtc/dtc`
用法为：
编译： `/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel/scripts/dtc/dtc -I dts -O dtb -o my_device_tree.dtb my_device_tree.dts`
反编译：`/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel/scripts/dtc/dtc -I dtb -O dts -o my_device_tree1.dts my_device_tree.dtb`

### 基本语法

```dts
/dts-v1/;
/ {
    model = "this is my device tree";
    #address-cells = <1>;
    #size-cells = <1>;
    node1{
        node-child{

        };
    };
    node2{
        node-child{

        };
    };
    LED:gpio@FDD60004{
        compatible = "LED";
        reg = <0xFDD60004 0x40>;
        status = "okay";
    };
};
```

1. 设备树**根节点**是整个设备树的起始点和顶层节点。 根节点以/标识符作为开头， 以{}表示根节点所包含的内容， 以分号表示结尾，

   ```text
   /dts-v1/; //设备树版本信息， 版本信息可选
   / {
       //根节点开始
       /*
           在这里可以添加注释， 描述根节点的属性和配置
       */
   };
   ```

2. 子节点
3. reg属性
4. model属性
5. status属性
6. compatible属性
7. aliases节点
8. chosen节点

### 中断

### 时钟

### CPU

在设备树中， cpu 的布局 cpus、 cpu-map、 socket、 cluster、 core 和 thread 等一系列节点描述。
cpu的层次结构就是cpus、 cpu-map、 socket、 cluster、 core 和 thread。

### 在实际工程中编写一个LED的设备树节点

首先，正点原子所有的设备树文件都保存在`/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel/arch/arm64/boot/dts/rockchip`目录下，如果想修改，就在这个目录下寻找，现在已知相关文件有`rk3568-atk-evb1-ddr4-v10.dtsi`、`rk3568-evb.dtsi`。

其中，`rk3568-evb.dtsi`为**核心板/通用功能级，尽量不改** ，其定义了核心板上的电源管理芯片（PMIC）、内存配置，或者 Linux 系统的通用配置。

而`rk3568-atk-evb1-ddr4-v10.dtsi`为正点原子板级配置，正点原子（ATK）针对自己推出的开发板所写的**顶层设备树文件**。它包含了前面的所有 `.dtsi`，并定义了底板上的具体接口（如哪个引脚接了按键、哪个接了 LED、屏幕型号等），**可以修改**。

目前做的修改：在`rk3568-evb.dtsi`注释掉了和`leds`相关的节点。同时，在`rk3568-atk-evb1-ddr4-v10.dtsi`里添加了有关led的设备树节点和pinctrl节点。

```dts
&pinctrl{
    rk_led {
        rk_led_gpio: rk-led-gpio {
            rockchip,pins =
                <0 RK_PC0 RK_FUNC_GPIO &pcfg_pull_up>;
        };
    };
};

myled: led {
    compatible = "topeet,led";
    pinctrl-names = "default";
    pinctrl-0 = <&rk_led_gpio>;
    gpios = <&gpio0 RK_PC0 GPIO_ACTIVE_LOW>;
};
```

修改完成后，在`/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/`目录下执行`./build.sh kernel`命令编译内核，无报错，待烧录验证。

> 如果不知道怎么写设备树，可以去`/home/alientek/RK3568/rk3568_linux_sdk/linux_sdk/kernel/Documentation/devicetree/bindings`目录下参考各个厂商的txt文件进行编写。

### 设备树展开流程

设备树会被编译成 DTB 文件并不会被编译进内核。 但是因为设备树描述的是硬件资源， 内核又要使用设备树中描述的硬件资源。 所以 Linux 会把设备树转换成内核可以识别的格式。
![设备树展开流程]({{ '/assets/img/blog/linux-driver-development/device-tree-unflatten-flow.png' | relative_url }})

#### dtb文件展开为device_node

内核在初始化的时候会将加载到内存里面的 dtb 文件展开成内核可以识别的设备树， 这里的可识别格式就是指将 dtb 解析成内核中的设备节点， 在 Linux 中用 device_node结构体表示设备节点。 device_node 结构体定义在内核源码的 include/linux/of.h 文件中。

```c
struct device_node {
    const char *name;
    const char *type;
    phandle phandle;
    const char *full_name;
    struct fwnode_handle fwnode;
    struct    property *properties;
    struct    property *deadprops;    /* removed properties */
    struct    device_node *parent;
    struct    device_node *child;
    struct    device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
    struct    kobject kobj;
#endif
    unsigned long _flags;
    void    *data;
#if defined(CONFIG_SPARC)
    const char *path_component_name;
    unsigned int unique_id;
    struct of_irq_controller *irq_trans;
#endif
};
```

可以通过 device_node 结构体中节点的名字、 父节点、 子节点和同级节点， 以代码方式构建出与 dts 文件中一致的设备节点关系。
其中 properties 字段是指向设备节点属性的指针。 设备节点的属性包含了与设备节点相关联的配置和参数信息。 property 字段同样定义在内核源码的 include/linux/of.h 文件中.

```c
struct property {
    char    *name;
    int    length;
    void    *value;
    struct property *next;
#if defined(CONFIG_OF_DYNAMIC) || defined(CONFIG_SPARC)
    unsigned long _flags;
#endif
#if defined(CONFIG_OF_PROMTREE)
    unsigned int unique_id;
#endif
#if defined(CONFIG_OF_KOBJ)
    struct bin_attribute attr;
#endif
};
```

![设备树节点转换]({{ '/assets/img/blog/linux-driver-development/device-tree-node-conversion.png' | relative_url }})
在内核中实际的流程如下图所示:
![设备树内核处理流程]({{ '/assets/img/blog/linux-driver-development/device-tree-kernel-flow.png' | relative_url }})

#### device_node 转换成platform_device

在平台总线模型中， device 部分是用 platform*device 结构体来描述硬件资源的， 所以内核最终会将内核认识的 device_node 树转换为 platform* device， 但是并不是所有的 device*node都会被转换成 platform* device， 只有满足要求的才会转换成 platform\_ device。

> 规则 1： 根节点下有 compatible 属性的子节点。
> 规则 2： 节点中 compatible 属性值是 simple-bus、 simple-mfd 和 isa 其中之一， 并且该节点下的子节点有 compatible 属性。 该节点下的子节点会被转换成 platform_device。
> 规则3： 节点中compatible属性有arm或primecell， 则对应的节点不会被转换成platform_device。

### 使用of函数获取设备树节点

用到了很多函数都可以获取设备树的节点，不一一展开，只给出用法，需要时可以Google或者查手册。

```c
const struct device_node *mydevicenode;

const struct of_device_id of_match_table[] = {
    { .compatible = "mydevicetree", },
    {}
};

const struct of_device_id *of_match_id;

int mydriver_probe(struct platform_device *dev){
    printk("this is mydevicetree_probe\n");

    mydevicenode = of_find_node_by_name(NULL, "myled");
    printk("By name mydevicenode->name is %s\n", mydevicenode->name);

    mydevicenode =  of_find_node_by_path("/topeet/myled");
    printk("By path mydevicenode->name is %s\n", mydevicenode->name);

    mydevicenode = of_get_parent(mydevicenode);
    printk("Parent mydevicenode->name is %s\n", mydevicenode->name);

    mydevicenode = of_get_next_child(mydevicenode, NULL);
    printk("First child mydevicenode->name is %s\n", mydevicenode->name);

    mydevicenode = of_find_compatible_node(NULL, NULL, "mydevicetree");
    printk("Compatible mydevicenode->name is %s\n", mydevicenode->name);

    mydevicenode = of_find_matching_node_and_match(NULL, of_match_table, &of_match_id);
    printk("Matching mydevicenode->name is %s\n", mydevicenode->name);
    printk("Matching of_match_id->compatible is %s\n", of_match_id->compatible);

    return 0;
}
```

### 使用of函数获取设备树属性

```c
struct resource *myresources;
struct property *myproperty;
int size;
int reg_size;
u32 reg_value32;
u64 reg_value64;
u32 reg_values32[2];
const char *compatible_value;

int mydriver_probe(struct platform_device *dev){
    struct device_node *mydevicenode = NULL;
    printk("-----------------------------------------------------------------\n");
    printk("this is mydevicetree_probe\n");
    // 1. 通过名称查找
    mydevicenode = of_find_node_by_name(NULL, "myled");
    if (mydevicenode) {
        printk("By name mydevicenode->name is %s\n", mydevicenode->name);
    } else {
        printk("Failed to find node by name 'myled'\n");
        return -ENODEV;
    }
    myproperty = of_find_property(mydevicenode, "compatible", &size);
    printk("myproperty->name is %s, size is %d\n", myproperty->name, size);

    reg_size = of_property_count_elems_of_size(mydevicenode, "reg", sizeof(__u32));
    printk("reg_size is %d\n", reg_size);

    of_property_read_u32_index(mydevicenode, "reg", 0, &reg_value32);
    of_property_read_u64_index(mydevicenode, "reg", 0, &reg_value64);
    printk("reg_value32 is %x, reg_value64 is %llx\n", reg_value32, reg_value64);
   
    of_property_read_variable_u32_array(mydevicenode, "reg", reg_values32, 1, 2);
    printk("reg_values32[0] is %x, reg_values32[1] is %x\n", reg_values32[0], reg_values32[1]);

    of_property_read_string(mydevicenode, "compatible", &compatible_value);
    printk("compatible is %s\n", compatible_value);
   
    of_node_put(mydevicenode);
    mydevicenode = NULL;
    printk("-----------------------------------------------------------------\n");
    return 0;
}
```

为什么使用`platform_get_resource(dev, IORESOURCE_MEM, 0);`会出错？ 需要添加ranges属性。**当没有 ranges 属性时， 代表这个设备只能被父节点访问。**

### ranges属性

ranges 属性用于描述子设备地址空间如何映射到父设备地址空间。 设备树中的每个设备节点都可以具有 ranges 属性， 其中包含了地址映射的信息。

```dts
ranges = <child-bus-address parent-bus-address length>;
```

child-bus-address：子设备地址空间的起始地址 。具体的字长由 ranges 所 在 节 点的 `#address-cells` 属性决定。
parent-bus-address： 父设备地址空间的起始地址。 具体的字长由 ranges 所在节点的父节点的 `#address-cells` 属性决定。
length： 映射的大小。 它指定了子设备地址空间在父设备地址空间中的长度。 具体的字长由 ranges 所在节点的父节点的 `#size-cells` 属性决定。
当 ranges 属性的值为空时， 表示子设备地址空间和父设备地址空间具有完全相同的映射，即 1:1 映射。
**当没有 ranges 属性时， 代表这个设备只能被父节点访问。**

### of获取中断资源

```c
int mydriver_probe(struct platform_device *dev){
    struct device_node *mydevicenode = NULL;
    printk("-----------------------------------------------------------------\n");
    printk("this is mydevicetree_probe\n");
   
    // 1. 通过名称查找
    mydevicenode = of_find_node_by_name(NULL, "myled");
    if (mydevicenode) {
        printk("By name mydevicenode->name is %s\n", mydevicenode->name);
    } else {
        printk("Failed to find node by name 'myled'\n");
        return -ENODEV;
    }
   
    myproperty = of_find_property(mydevicenode, "compatible", &size);
    printk("myproperty->name is %s, size is %d\n", myproperty->name, size);

    reg_size = of_property_count_elems_of_size(mydevicenode, "reg", sizeof(__u32));
    printk("reg_size is %d\n", reg_size);

    of_property_read_u32_index(mydevicenode, "reg", 0, &reg_value32);
    of_property_read_u64_index(mydevicenode, "reg", 0, &reg_value64);
    printk("reg_value32 is %x, reg_value64 is %llx\n", reg_value32, reg_value64);

    of_property_read_variable_u32_array(mydevicenode, "reg", reg_values32, 1, 2);
    printk("reg_values32[0] is %x, reg_values32[1] is %x\n", reg_values32[0], reg_values32[1]);

    of_property_read_string(mydevicenode, "compatible", &compatible_value);
    printk("compatible is %s\n", compatible_value);

    //platform_get_resource(dev, IORESOURCE_MEM, 0);
    of_node_put(mydevicenode);
    mydevicenode = NULL;
    printk("-----------------------------------------------------------------\n");
    return 0;
}
```

## 设备树插件

设备树插件是为了提升效率，不需要每次更改完设备树后都重新编译烧录内核。

```dts
//第一种写法
&{/rm500u-5g}{
    overlay_node {
        status = "okay";
    };
};
```

```dts
//第二种写法

&rm500u_5g{
    over_node {
        status = "okay";
    };
};
```

```dts
//第三种写法
/{
    fragments@0 {
        target-path = "/rm500u-5g";
        __overlay__ {
            status = "okay";
        };
    };
   
    fragments@1 {
        target = <&rm500u_5g>;
        __overlay__ {
            status = "okay";
        };
    };
};
```

### 为什么设备树插件的实现使用了configfs而不是sysfs？

[[sysfs和configfs的对比和设备树插件的选择]]

### configfs 相关核心数据结构

![configfs 核心结构]({{ '/assets/img/blog/linux-driver-development/configfs-core-structures.png' | relative_url }})

## 设备模型

设备模型包括总线（bus）、设备（device）、驱动（driver）和类（class）四个概念。
其中比较关键的点有虚拟总线 platform，设备目录 `/sys/device` 和类目录 `sys/class` 等，这些目录下的内容都是对 device 目录下的软连接。

### kobject 和 kset 框架

kobject 是 `/sys` 下的一个目录，为了让设备驱动模型中的多个对象以层次分明、 逻辑清晰的方式组织在一起， Linux 使用 kset 结构体作为 kobject 的容器。 每个 kset 也会在/sys 目录下对应一个子目录。实现在一个目录（kset） 下包含多个目录（kobject）。

```c
struct kobject {
    const char *name; //表示 kobject 的名称， 通常用于在/sys 目录下创建对应的目录struct
    list_headentry; //用于将 kobject 链接到父 kobject 的子对象列表中， 以建立层次关系。
    struct kobject *parent; //指向父 kobject， 表示 kobject 的层次关系。
    struct kset *kset; //指向包含该 kobject 的 kset， 用于进一步组织和管理 kobject。
    struct kobj_type *ktype; //指向定义 kobject 类型的 kobj_type 结构体， 描述 kobject 的属性和操作。
    struct kernfs_node *sd; //指向 sysfs 目录中对应的 kernfs_node
    struct kref kref; //用于对 kobject 进行引用计数， 确保在不再使用时能够正确释放资源。
 }
```

```c
struct kset {
    struct list_head list; spinlock_t list_lock;
    struct kobject kobj;//包含 kobject,说明 kset 也会在 sys 目录下对应一个目录
    const struct kset_uevent_ops *uevent_ops;
}
```

![kobject 层级关系]({{ '/assets/img/blog/linux-driver-development/kobject-hierarchy.png' | relative_url }})
![kobject 与 sysfs 布局]({{ '/assets/img/blog/linux-driver-development/kobject-sysfs-layout.png' | relative_url }})

### 创建 kobject

```c
static int kobj_init(void){

    int ret;

    kobj01 = kobject_create_and_add("kobj01", NULL);

    kobj02 = kobject_create_and_add("kobj02", kobj01);

    kobj03 = kzalloc(sizeof(kobj03), GFP_KERNEL);

    ret = kobject_init_and_add(kobj03, &myktype, NULL, "%s", "kobj03");

    return 0;

}


static void kobj_exit(void){

    kobject_put(kobj03);

    kobject_put(kobj02);

    kobject_put(kobj01);

}
```

### 创建 kset

```c
static int kobj_init(void){

    int ret;

    kset01 = kset_create_and_add("kset01", NULL, NULL);


    kobj01 = kzalloc(sizeof(struct kobject), GFP_KERNEL);

    kobj01->kset = kset01;

    ret = kobject_init_and_add(kobj01, &myktype, NULL, "%s", "kobj01");


    kobj02 = kzalloc(sizeof(struct kobject), GFP_KERNEL);

    kobj02->kset = kset01;

    ret = kobject_init_and_add(kobj02, &myktype, NULL, "%s", "kobj02");

    return 0;
}



static void kobj_exit(void){

    kobject_put(kobj02);

    kobject_put(kobj01);

    kset_unregister(kset01);

}
```

### 创建文件属性

### 注册自定义总线

### 在总线下注册设备

在总线下注册设备主要使用 `device_register()` 函数，需要完善 `device` 设备结构体。

```c
extern struct bus_type rk_bus;

void    device_release(struct device *dev){
    printk("this is device_release\n");
}

struct device device1 = {
    .init_name = "bus_device1",
    .bus = &rk_bus,
    .release = device_release,
    .devt = ((255<<20|0)),
};

static int bus_device_init(void){
    int ret;
    ret = device_register(&device1);

    return ret;
}

static void bus_device_exit(void){
    device_unregister(&device1);
}
```

## 热插拔

热插拔是指在不关闭系统电源的情况下， 动态插入或拔出硬件设备的能力。
热插拔机制有 `udev` 和 `mdev` 等，嵌入式常用 `mdev`，X86 上常用 `udev`。

udev 是基于 netlink 实现的。 netlink 是一种特殊的套接字类型， 用于内核空间和用户空间之间的通信。 当内核发生 uevent 事件时， 会通过 netlink 将该事件发送给用户空间的 udevd 守护进程。 udevd 守护进程负责接收来自内核的事件， 并按照预定义的规则文件（ 通常位于/etc/udev/rules.d/） 决定如何处理这些事件， 如创建或删除设备节点、 设置设备权限等。
mdev 则是基于 uevent_helper 机制来处理设备的插入和拔出事件。 当内核产生 uevent 时，会调用 uevent_helper 所指定的用户程序来执行热插拔操作。 系统启动时， 可以通过命令 echo /sbin/mdev >/proc/sys/kernel/hotplug 指定 mdev 为处理程序。 这样当内核产生 uevent 时， 会自动调用 mdev 并按照预定义的规则文件（通常位于/etc/mdev.conf） 执行对应的热拔插动作。

### kobject_uevent 函数

kobject_uevent()函数是 Linux 内核用于生成和发送 uevent 事件的 API 函数。

```c
int kobject_uevent(struct kobject kobj, enum kobject_action action);
```

kobject_uevent()函数的主要作用是在内核中生成 uevent 事件， 并通过 netlink 机制将该事件发送给用户空间的 udev。 在调用该函数时， 内核会将相关的设备信息和事件类型封装为 uevent 消息， 并通过 netlink 套接字将消息发送给用户空间。

### udevadm 命令

udevadm 是一个用于与 udev 进行交互的命令行工具。 用于查询和管理设备、 触发 uevent 事件以及执行其他与 udev 相关的操作。 一些常见的 udevadm 命令及其功能如下：

1. udevadm info 命令： 用于获取设备的详细信息， 包括设备路径、 属性、 驱动程序等。
2. udevadm monitor 命令： 用于监视和显示当前系统中的 uevent 事件。 它会实时显示设备的插入、 拔出以及其他相关事件。
3. udevadm trigger 命令： 用于手动触发设备的 uevent 事件。 可以使用该命令模拟设备的插入、 拔出等操作， 以便触发相应的事件处理。
4. udevadm settle 命令： 用于等待 udev 处理所有已排队的 uevent 事件。 它会阻塞直到 udev 完成当前所有的设备处理操作。
5. udevadm control 命令： 用于与 udev 守护进程进行交互， 控制其行为。 例如， 可以使用该命令重新加载 udev 规则、 设置日志级别等。
6. udevadm test 命令： 用于测试 udev 规则的匹配和执行过程。 可以通过该命令测试特定设备是否能够正确触发相应的规则。
7. 在开发板上烧写 buildroot 系统， 输入 udevadm monitor &命令可以监视和显示当前系统中的 uevent 事件

### kset_event_ops 结构体

`kset_event_ops` 是 Linux 内核设备模型中一个核心的回调函数集合。它的作用是让一个 `kset`（kobject 的集合）能够控制其下所有 `kobject` 在发生状态变化（如添加、移除）时的行为，尤其是如何与用户空间进行“热插拔”（uevent）通信。

它主要包含三个函数，在 `kobject_uevent()` 被调用时按特定流程执行[](https://blog.csdn.net/yukonjian/article/details/79233859)[](https://developer.aliyun.com/article/1273931)。各函数的作用和执行位置如下：

| 函数名       | 主要作用                                                                                                                          | 执行位置（调用时机）                                                                 |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **`filter`** | **过滤器**：决定是否允许某个 `kobject` 的 uevent 事件发送到用户空间。如果返回 `0`，该事件将被忽略，不再发送。                     | 在 `kobject_uevent_env()` 函数执行的早期阶段被调用。                                 |
| **`name`**   | **提供子系统名**：返回一个字符串，作为该 `kobject` 所属子系统的名称。这个名称会作为 `SUBSYSTEM=` 环境变量的一部分传递给用户空间。 | 在 `filter` 检查通过之后，准备环境变量之前被调用。                                   |
| **`uevent`** | **自定义环境变量**：允许 `kset` 向 uevent 消息中添加自定义的环境变量（key=value 对），补充更多信息。                              | 在环境变量缓冲区（`env`）创建好，并填入了 `ACTION`、`DEVPATH` 等默认字段之后被调用。 |

内核中 `kobject_uevent_env()` 函数（位于 `lib/kobject_uevent.c`）的核心逻辑，这三个函数就是在这里被调用的：

1. **查找所属 kset**：首先，内核会沿着 `kobject` 的父子链向上查找，直到找到一个属于某个 `kset` 的祖先 `kobject`，并获取这个 `kset` 及其 `uevent_ops`。

```c
/* search the kset we belong to */
top_kobj = kobj;
while (!top_kobj->kset && top_kobj->parent)
    top_kobj = top_kobj->parent;

if (!top_kobj->kset) {
    pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
         "without kset!\n", kobject_name(kobj), kobj,
         __func__);
    return -EINVAL;
}

kset = top_kobj->kset;
uevent_ops = kset->uevent_ops;
```

2. **执行 filter**：紧接着会调用 `uevent_ops->filter(kset, kobj)`。如果这个函数存在且返回 `0`，整个 uevent 发送过程会立即终止，事件被“过滤”掉。

```c
/* skip the event, if the filter returns zero. */
if (uevent_ops && uevent_ops->filter)
    if (!uevent_ops->filter(kset, kobj)) {
        pr_debug("kobject: '%s' (%p): %s: filter function "
             "caused the event to drop!\n",
             kobject_name(kobj), kobj, __func__);
        return 0;
    }
```

3. **执行 name**：然后调用 `uevent_ops->name(kset, kobj)` 获取子系统名。如果该函数指针为 `NULL`，则默认使用 `kset` 自身的 `kobject` 名字作为子系统名。

```c
/* originating subsystem */
if (uevent_ops && uevent_ops->name)
    subsystem = uevent_ops->name(kset, kobj);
else
    subsystem = kobject_name(&kset->kobj);
if (!subsystem) {
    pr_debug("kobject: '%s' (%p): %s: unset subsystem caused the "
         "event to drop!\n", kobject_name(kobj), kobj,
         __func__);
    return 0;
}
```

3. **执行 uevent**：在构建好 uevent 消息的基本环境变量（如 `ACTION=add`、`DEVPATH=/…`、`SUBSYSTEM=…`）后，内核会调用 `uevent_ops->uevent(kset, kobj, env)`，让 `kset` 有机会通过 `add_uevent_var()` 宏向 `env` 中添加自己的环境变量。

```c
/* default keys */
retval = add_uevent_var(env, "ACTION=%s", action_string);
if (retval)
    goto exit;
retval = add_uevent_var(env, "DEVPATH=%s", devpath);
if (retval)
    goto exit;
retval = add_uevent_var(env, "SUBSYSTEM=%s", subsystem);
if (retval)
    goto exit;

/* let the kset specific function add its stuff */
if (uevent_ops && uevent_ops->uevent) {
    retval = uevent_ops->uevent(kset, kobj, env);
    if (retval) {
        pr_debug("kobject: '%s' (%p): %s: uevent() returned "
             "%d\n", kobject_name(kobj), kobj,
             __func__, retval);
        goto exit;
    }
}
```

### netlink

内核通过 kobject uevent 接口发送广播事件之后， 用户空间可以通过 netlink 来监听这些广播信息。 通过监听广播信息， 就可以获取到携带环境变量的事件。
在 linux 中，netlink 是通过 socket 实现的。流程是 `创建socket-> bind绑定套接字 -> recv接收数据`。
因为很多都是固定用法，在此不展开每个函数的变量了，仅给出可运行的 netlink 代码，需要时 Google。

```c
#include <strings.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <linux/netlink.h>
#include <string.h>

//netlink 连接的流程是 创建 socket-> 绑定bind-> 接收recv
int main(int argc, char **argv){
    int socket_fd;
    int ret=0;
    int len=0;
    int i = 0;
    struct sockaddr_nl nl;
    char buf[4096] = {0};

    bzero(&nl,sizeof(struct sockaddr_nl));
    nl.nl_family = AF_NETLINK;
    nl.nl_groups = 1;
    nl.nl_pid = 0;

    socket_fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_KOBJECT_UEVENT);
    if(socket_fd < 0){
        printf("socket error \n");
        return -1;
    }

    ret = bind(socket_fd, (struct sockaddr*)&nl, sizeof(struct sockaddr_nl));
    if(ret< 0){
        printf("bind error \n");
        return -1;
    }

    while(1) {
    memset(buf, 0, sizeof(buf));
        len = recv(socket_fd, buf, sizeof(buf), 0);
        if (len <= 0)
            break;

        char *pos = buf;
        while (pos < buf + len) {
            printf("%s\n", pos);  // 每个字符串单独一行
            pos += strlen(pos) + 1;  // 跳过当前字符串和 \0
    }
}
    return 0;
}

```
