---
title: "面试算法题总结"
date: 2018-04-21T21:35:04+08:00
lastmod: 2018-04-21T21:35:04+08:00
draft: true
keywords: []
description: ""
tags: []
categories: ["algorithm"]
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: true
postMetaInFooter: false
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

本文用于记录总结面试中常见的算法题。

<!--more-->

## 001
### 题目描述

对于一个字节(`8bit`)的无符号整形变量，求二进制表示中`1`的个数，要求算法执行效率尽可能地高。

### 方法1

```c
#include <stdio.h>

unsigned char count(unsigned char byte)
{
	unsigned char num = 0;
	while(byte) {
		num += (byte & 0x01);
		byte >>= 1;
	}
	return num;
}

int main(int argc, char **argv)
{
	printf("number of 1 bit in 80 is %d\n", (int)count(80));
	printf("number of 1 bit in 92 is %d\n", (int)count(92));
}
```

不管有多少个1都要循环8次，执行效率不高，但是执行该函数的时间每次都是确定的

### 方法2

除以`2`向右移位， 逐个统计，但是用到取模和相除，这个很耗资源。

```c
#include <stdio.h>

unsigned char count(unsigned char byte)
{
	unsigned char num = 0;
	while(byte) {
		if (byte%2 == 1)
			num++;
		byte /= 2;
	}
	return num;
}

int main(int argc, char **argv)
{
	printf("number of 1 bit in 80 is %d\n", (int)count(80));
	printf("number of 1 bit in 92 is %d\n", (int)count(92));
}
```
### 方法3

使用位操作，但是只会去统计1的个数，循环的次数是BYTE中1的个数，无需遍历

```c
#include <stdio.h>

unsigned char count(unsigned char byte)
{
	unsigned char num = 0;
	while(byte) {
		//byte=byte&(byte-1)这个操作可以直接消除掉byte中的最右边的1。
		byte &= (byte-1);
		num++;
	}
	return num;
}

int main(int argc, char **argv)
{
	printf("number of 1 bit in 80 is %d\n", (int)count(80));
	printf("number of 1 bit in 92 is %d\n", (int)count(92));
}
```

循环次数与`byte`中`1`的个数有关，但是函数执行时间不确定，不过效率比前面的要提高了很多。


### 方法4
查表法，这个的效率应该是最高的了——空间换时间。将0~255各个数中所含的1列出来，查表！！


```c
#include <stdio.h>

int count_table[256] =
{
	0,1,1,2,1,2,2,3,1,2,2,3,2,3,3,4,
	1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,
	1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
	1,2,2,3,2,3,3,4,2,3,3,4,3,4,4,5,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
	2,3,3,4,3,4,4,5,3,4,4,5,4,5,5,6,
	3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
	3,4,4,5,4,5,5,6,4,5,5,6,5,6,6,7,
	4,5,5,6,5,6,6,7,5,6,6,7,6,7,7,8
};

unsigned char count(unsigned char byte)
{
	return count_table[byte];
}

int main(int argc, char **argv)
{
	printf("number of 1 bit in 80 is %d\n", (int)count(80));
	printf("number of 1 bit in 92 is %d\n", (int)count(92));
}
```

这种方法时间复杂度最低了`O(1)`，执行时间也是确定的，这个是典型的空间换时间。

## 002
