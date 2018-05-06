---
title: "x86_64内存管理寄存器"
date: 2018-05-06T21:23:25+08:00
lastmod: 2018-05-06T21:23:25+08:00
draft: true
keywords: ["memory", "registers", "x86_64"]
description: ""
tags: ["kernel", "linux", "memory management"]
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

本文详细介绍了`x86/x86_64`内存管理寄存器`GDTR`、`LDTR`、`IDTR`和`TR`。

<!--more-->

## 引入

我们先看一幅图，它详细展示了`IA-32e`模式下，系统级的寄存器和数据结构：

![enter description here][1]

本文会对上面标红的四个寄存器进行详细的说明。

![enter description here][2]


## 段寄存器

`IA-32e`模式下，有`6`个段寄存器，如下：

![enter description here][3]

每个段寄存器，分为可见部分和隐藏部分:

* 可见部分为`16位段选择子`，
* 隐藏部分为`16位段选择子对应的段的基地址（32bit）、段界限（20bit）和属性信息（12bit）`，它的长度为64位。

> 隐藏部分可以理解为段选择子对应的段描述符的缓存。

## 几个概念

### 全局描述符表`GDT（Global Descriptor Table`）

在整个系统中，全局描述符表`GDT`只有一张(*一个处理器对应一个GDT*)，`GDT`可以被放在内存的任何位置，但`CPU`必须知道`GDT`的入口，也就是基地址放在哪里，`Intel`的设计者门提供了一个寄存器`GDTR`用来存放`GDT`的入口地址，程序员将`GDT`设定在内存中某个位置之后，可以通过`LGDT`指令将`GDT`的入口地址装入此寄存器，从此以后，`CPU`就根据此寄存器中的内容作为`GDT`的入口来访问`GDT`了。`GDTR`中存放的是`GDT`在内存中的基地址和其表长界限。

基地址指定`GDT`表中字节0在线性地址空间中的地址，表长度指明`GDT`表的字节长度值。指令`LGDT`和`SGDT`分别用于加载和保存`GDTR`寄存器的内容。在机器刚加电或处理器复位后，基地址被默认地设置为`0`，而长度值被设置成`0xFFFF`。在保护模式初始化过程中必须给`GDTR`加载一个新值。


### 段选择子（Selector）

由`GDTR`访问全局描述符表是通过`段选择子`（实模式下的段寄存器）来完成的。段选择子是一个`16`位的寄存器:

![enter description here][4]

段选择子包括三部分：**描述符索引**（`index`）、**TI**、**请求特权级**（`RPL`）:

* `index`（描述符索引）部分表示所需要的段的描述符在描述符表的位置，由这个位置再根据在`GDTR`中存储的描述符表基址就可以找到相应的描述符。然后用描述符表中的段基址加上逻辑地址（`SEL:OFFSET`）的`OFFSET`就可以转换成线性地址，
* 段选择子中的`TI`值只有一位`0`或`1`，`0`代表选择子是在`GDT`选择，`1`代表选择子是在`LDT`选择。
* 请求特权级（`RPL`）则代表选择子的特权级，共有`4`个特权级（`0级、1级、2级、3级`）。

> 关于特权级的说明：任务中的每一个段都有一个特定的级别。每当一个程序试图访问某一个段时，就将该程序所拥有的特权级与要访问的特权级进行比较，以决定能否访问该段。系统约定，CPU只能访问同一特权级或级别较低特权级的段。

例如给出逻辑地址：21h:12345678h转换为线性地址

a. 选择子`SEL=21h=0000000000100 0 01b` 他代表的意思是：选择子的`index=4`即`100b`选择`GDT`中的第4个描述符；`TI=0`代表选择子是在`GDT`选择； 最后的`01b`代表特权级`RPL=1`

b. `OFFSET=12345678h`若此时`GDT`第四个描述符中描述的段基址（Base）为`11111111h`，则线性地址=`11111111h+12345678h=23456789h`


### 局部描述符表LDT（Local Descriptor Table）

局部描述符表可以有若干张，每个任务可以有一张。我们可以这样理解`GDT`和`LDT`：`GDT`为一级描述符表，`LDT`为二级描述符表。如图：

![enter description here][5]

`LDT`和`GDT`从本质上说是相同的，只是`LDT`嵌套在`GDT`之中。`LDTR`记录局部描述符表的起始位置，与`GDTR`不同，`LDTR`的内容是一个`段选择子`。由于`LDT`本身同样是一段内存，也是一个段，所以它也有个描述符描述它，这个描述符就存储在`GDT`中，对应这个表述符也会有一个选择子，`LDTR`装载的就是这样一个选择子。`LDTR`可以在程序中随时改变，通过使用`lldt`指令。如上图，如果装载的是`Selector 2`则`LDTR`指向的是表`LDT2`。

举个例子：如果我们想在表`LDT2`中选择第三个描述符所描述的段的地址`12345678h`。

1. 首先需要装载`LDTR`使它指向`LDT2` 使用指令`lldt`将`Select2`装载到`LDTR`
2. 通过逻辑地址（`SEL:OFFSET`）访问时`SEL`的`index=3`代表选择第三个描述符；`TI=1`代表选择子是在`LDT`选择，此时`LDTR`指向的是`LDT2`,所以是在`LDT2`中选择，此时的`SEL`值为`1Ch`(二进制为`11 1 00b`)。`OFFSET=12345678h`。逻辑地址为`1C:12345678h`
3. 由`SEL`选择出描述符，由描述符中的基址（`Base`）加上`OFFSET`可得到线性地址，例如基址是`11111111h`，则线性地址=`11111111h+12345678h=23456789h`
4. 此时若再想访问LDT1中的第三个描述符，只要使用`lldt`指令将选择子`Selector 1`装入再执行`2`、`3`两步就可以了（因为此时`LDTR`又指向了`LDT1`）

由于每个进程都有自己的一套程序段、数据段、堆栈段，有了局部描述符表则可以将每个进程的程序段、数据段、堆栈段封装在一起，只要改变`LDTR`就可以实现对不同进程的段进行访问。

当进行任务切换时，处理器会把新任务`LDT`的段选择符和段描述符自动地加载进`LDTR`中。在机器加电或处理器复位后，段选择符和基地址被默认地设置为`0`，而段长度被设置成`0xFFFF`。

## 实例

### 访问GDT


![enter description here][6]

当`TI=0`时表示段描述符在`GDT`中，如上图所示：

1. 先从`GDTR`寄存器中获得`GDT`基址。
2. 然后再`GDT`中以段选择器高13位位置索引值得到段描述符。
3. 段描述符符包含段的基址、限长、优先级等各种属性，这就得到了段的起始地址（基址），再以基址加上偏移地址`yyyyyyyy`才得到最后的线性地址。

### 访问LDT

![enter description here][7]

当`TI=1`时表示段描述符在`LDT`中，如上图所示：

1. 还是先从`GDTR`寄存器中获得`GDT`基址。
2. 从`LDTR`寄存器中获取`LDT`所在段的位置索引(`LDTR`高13位)。
3. 以这个位置索引在`GDT`中得到`LDT`段描述符从而得到`LDT`段基址。
4. 用段选择器高13位位置索引值从`LDT`段中得到段描述符。
5. 段描述符符包含段的基址、限长、优先级等各种属性，这就得到了段的起始地址（基址），再以基址加上偏移地址`yyyyyyyy`才得到最后的线性地址。


### GDT and LDT

intel 手册上对`GDT`和`LDT`的图示如下：

![enter description here][8]


## 扩展

### 中断描述符表寄存器IDTR 

与`GDTR`的作用类似，`IDTR`寄存器用于存放中断描述符表`IDT`的32位**线性基地址**和16位**表长度**值。指令`LIDT`和`SIDT`分别用于加载和保存`IDTR`寄存器的内容。在机器刚加电或处理器复位后，基地址被默认地设置为`0`，而长度值被设置成`0xFFFF`。

### 任务寄存器TR

`TR`用于寻址一个特殊的任务状态段（Task State Segment，TSS）。`TSS`中包含着当前执行任务的重要信息。

`TR`寄存器用于存放当前任务`TSS段`的**16位段选择符**、**32位基地址**、**16位段长度**和**描述符属性值**。它引用`GDT`表中的一个`TSS`类型的描述符。指令`LTR`和`STR`分别用于加载和保存`TR`寄存器的段选择符部分。当使用`LTR`指令把选择符加载进任务寄存器时，`TSS`描述符中的段基地址、段限长度以及描述符属性会被自动加载到任务寄存器中。当执行任务切换时，处理器会把新任务的`TSS`的段选择符和段描述符自动加载进任务寄存器`TR`中。


## 参考文档

http://www.techbulo.com/708.html

  [1]: ./system-level-registers-and-data-structures-in-IA-32e-mode.png "system-level-registers-and-data-structures-in-IA-32e-mode"
  [2]: ./memory-management-register.png "memory-management-register"
  [3]: ./segment-registers.png "segment-registers"
  [4]: ./segment-selector.png "segment-selector"
  [5]: ./gdt-ldt.jpeg "gdt-ldt"
  [6]: ./GDT.jpeg "GDT"
  [7]: ./LDT.jpeg "LDT"
  [8]: ./global-and-local-descriptor-tables.png "global-and-local-descriptor-tables"
