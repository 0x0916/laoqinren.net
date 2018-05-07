---
title: "内核基础设施——wait queue"
date: 2018-05-07T18:27:48+08:00
lastmod: 2018-05-07T18:27:48+08:00
draft: true
keywords: ["等待队列"]
description: ""
tags: ["kernel", "linux", "wait queue"]
categories: ["kernel"]
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
postMetaInFooter: true
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false

flowchartDiagrams:
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

## 介绍：什么是等待队列？

在软件开发中任务经常由于某种条件没有得到满足而不得不进入睡眠状态，然后等待条件得到满足的时候再继续运行，进入运行状态。这种需求需要等待队列机制的支持。`Linux`中提供了等待队列的机制，该机制在内核中应用很广泛。
 
等待队列实现了在事件上的条件等待：希望等待特定事件的进程将自己放进合适的等待队列中，并放弃控制权。因此，等待队列表示一组睡眠的进程，当某一条件变为真时，由内核唤醒他们。
 
在`Linux`内核中使用等待队列的过程很简单，首先定义一个`wait_queue_head`，然后如果一个`task`想等待某种事件，那么调用`wait_event（等待队列，事件）`就可以了。

> 本文中使用的内核版本为：[3.10.0-693.21.1](http://vault.centos.org/7.4.1708/updates/Source/SPackages/kernel-3.10.0-693.21.1.el7.src.rpm)

<!--more-->

## 初始化等待队列

### 静态初始化

```c
DECLARE_WAIT_QUEUE_HEAD(name)
```

### 动态初始化

```c
wait_queue_head_t q;
init_waitqueue_head(&q);
```

## 将进程加入等待队列

|API接口|说明|
|---|---|
|wait_event(wq, condition)|将当前进程加入等待队列wq中，并设置进程状态为D，然后睡眠直到condition为true|
|wait_event_timeout(wq, condition, timeout)|将当前进程加入等待队列wq中，并设置进程状态为D，然后睡眠直到condition为true,即使condition不为true，如果超时后，也结束睡眠状态|
|wait_event_cmd(wq, condition, cmd1, cmd2)|跟wait_event类似，只不过在睡眠前执行cmd1，睡眠后执行cmd2|
|wait_event_interruptible(wq, condition)|跟wait_event类似，只不过设置进程的状态为S|
|wait_event_interruptible_timeout(wq, condition, timeout)|跟wait_event_timeout 类似，只不过设置进程的状态为S|
|wait_event_killable(wq, condition)|跟wait_event类似,只不过睡眠后，可以接受信号|

## 唤醒等待队列中的进程

|API接口|说明|
|---|---|
|wake_up|唤醒等待队列上的一个进程，考虑TASK_INTERRUPTIBLE 和TASK_UNINTERRUPTIBLE|
|wake_up_nr|唤醒等待队列上的nr个进程，考虑TASK_INTERRUPTIBLE 和TASK_UNINTERRUPTIBLE|
|wake_up_all|唤醒等待队列上的所有进程，考虑TASK_INTERRUPTIBLE 和TASK_UNINTERRUPTIBLE|
|wake_up_locked|跟wake_up类似，只是调用时已经确保加锁了|
|wake_up_all_locked|跟wake_up_all类似，只是调用时已经确保加锁了|
|wake_up_interruptible|唤醒等待队列上的一个进程，只考虑TASK_INTERRUPTIBLE|
|wake_up_interruptible_nr|唤醒等待队列上的nr个进程，只考虑TASK_INTERRUPTIBLE|
|wake_up_interruptible_all|唤醒等待队列上的所有进程，只考虑TASK_INTERRUPTIBLE|
|wake_up_interruptible_sync||


## 示例程序

首先，我解释一下示例程序。

在示例程序中，有两个地方唤醒等待队列中的进程，一处是在`read `函数（`cat /proc/test_wait_queue`）中，另一处在模块退出函数中。

在模块初始化函数中，我们创建了一个内核线程（`MyWaitThread`）,该内核线程总是等待`wait_queue_flag`不为0，它一直睡眠，直到有人唤醒它。当唤醒后，它检查wait_queue_flag的值，如果是1，说明唤醒来自read函数，此时它打印read count 信息，并继续睡眠，如果wait_queue_flag的值是2，说明唤醒来自模块退出函数，此时给内核线程退出。

代码如下：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/wait.h>
#include <linux/kthread.h>

static int read_count = 0;
static struct task_struct *wait_thread;
// Initializing waitqueue statically
DECLARE_WAIT_QUEUE_HEAD(test_waitqueue);
static int wait_queue_flag = 0;

static int my_waitqueue_show(struct seq_file *m, void *v)
{
        printk(KERN_ALERT "Read function\n");
        seq_printf(m, "read_count = %d\n", read_count);
        wait_queue_flag = 1;
        wake_up_interruptible(&test_waitqueue); // wake up only one process from wait queue
        return 0;
}

static int my_waitqueue_open(struct inode *inode, struct file *filp)
{
        return single_open(filp, my_waitqueue_show, NULL);
}

static struct file_operations test_wait_queue_fops = {
        .open           = my_waitqueue_open,
        .read           = seq_read,
        .llseek         = seq_lseek,
        .release        = single_release,
};

static int wait_function(void *unused)
{
        while(1) {
                printk(KERN_ALERT "Waiting For Event...\n");
                // sleep until wait_queue_flag != 0
                wait_event_interruptible(test_waitqueue, wait_queue_flag != 0);
                if (wait_queue_flag == 2) {
                        printk(KERN_ALERT "Event Came From Exit Function\n");
                        return 0;
                }
                printk(KERN_ALERT "Event Came From Read Function - %d\n", ++read_count);
                wait_queue_flag = 0;
        }

        return 0;
}

static int __init mywaitqueue_init(void)
{
        struct proc_dir_entry *pe;

        printk(KERN_ALERT "[Hello] mywaitqueue \n");
        pe = proc_create("test_wait_queue", 0644, NULL, &test_wait_queue_fops);
        if (!pe)
                return -ENOMEM;

        // Create the kernel thread with name "MyWaitThread"
        wait_thread = kthread_create(wait_function, NULL, "MyWaitThread");
        if (wait_thread) {
                printk(KERN_ALERT "Thread created successfully\n");
                wake_up_process(wait_thread);
        } else {
                printk(KERN_ALERT "Thread creation failed\n");
        }

        return 0;
}

static void __exit mywaitqueue_exit(void)
{
        wait_queue_flag = 2;
        wake_up_interruptible(&test_waitqueue);
        printk(KERN_ALERT "[Goodbye] mywaitqueue\n");
        remove_proc_entry("test_wait_queue", NULL);
}

module_init(mywaitqueue_init);
module_exit(mywaitqueue_exit);
MODULE_LICENSE("GPL");

```

Makefile如下：

```makefile
ifneq ($(KERNELRELEASE), )
obj-m := mywaitqueue.o
else
KERNELDIR ?=/lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
all:
        $(MAKE) -C $(KERNELDIR) M=$(PWD) modules
clean:
        $(MAKE) -C $(KERNELDIR) M=$(PWD) clean 
endif
```

编译成功后，插入模块：
```bash
# insmod mywaitqueue.ko
```

检查dmesg

```
[167034.774139] [Hello] mywaitqueue 
[167034.774330] Thread created successfully
[167034.774393] Waiting For Event...
```

读两次

```bash
 # cat /proc/test_wait_queue 
read_count = 0
 # cat /proc/test_wait_queue 
read_count = 1 
 ```
 
 查看dmesg
 ```
[167047.972321] Read function
[167047.972387] Event Came From Read Function - 1
[167047.972390] Waiting For Event...
[167049.779541] Read function
[167049.779615] Event Came From Read Function - 2
[167049.779618] Waiting For Event...
```

卸载模块`rmmod mywaitqueue`,dmesg如下：

```
[167110.886314] [Goodbye] mywaitqueue
[167110.886317] Event Came From Exit Function
```


## Linux中等待队列的实现

等待队列应用广泛，但是内核实现却十分简单。其涉及到两个比较重要的数据结构：

### wait_queue_head

该结构描述了等待队列的链头，其包含一个链表和一个原子锁，结构定义如下：

```c
struct wait_queue_head {
        spinlock_t              lock;
        struct list_head        head;
};
typedef struct wait_queue_head wait_queue_head_t;
```
###  wait_queue_entry

该结构是对一个等待任务的抽象。每个等待任务都会抽象成一个`wait_queue_entry`，并且挂载到`wait_queue_head`上。该结构定义如下：

```c
typedef struct wait_queue_entry wait_queue_entry_t;

/*
 * A single wait-queue entry structure:
 */
struct wait_queue_entry {
        unsigned int            flags;
        void                    *private;
        wait_queue_func_t       func;
        struct list_head        entry;
};
```

`wait_event`用于将当前进程加入某一等待队列中，同时将该进程的状态修改为等待状态。而`wake_up`则用于将某一个等待队列上面所有的等待进程唤醒，也就是将其从等待队列上面删掉，同时将其的进程状态置为可运行状态。


等待队列的结构如下图所示:

![enter description here][1]


## 参考文档

https://embetronicx.com/tutorials/linux/device-drivers/waitqueue-in-linux-device-driver-tutorial/



  [1]: ./wait_queue.png "wait_queue"

