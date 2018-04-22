---
title: "Linux内存相关的调节参数"
date: 2018-04-22T18:56:39+08:00
lastmod: 2018-04-22T18:56:39+08:00
draft: true
keywords: []
description: ""
tags: ["kernel", "linux", "virtual memory", "tune"]
categories: ["linux"]
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: true
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

在`Linux`中，`/proc/sys/vm`下面的配置文件都跟`virtual memory (VM) subsystem`相关，本文详细分析了这些配置，并尝试给出最优化的建议。

> 注意，本文中的代码是基于稳定版本的内核`v4.4.128`。

<!--more-->

## admin_reserve_kbytes
## block_dump
## compact_memory
## dirty_background_bytes
## dirty_background_ratio
## dirty_bytes
## dirty_expire_centisecs
## dirty_ratio
## dirty_writeback_centisecs
## drop_caches

`drop_caches`用来手动清理`linux`系统中的缓存，包括`pagecache`和`slab objects(include dentries and inodes)`。

**释放pagecache**
```bash
root@localhost ~# echo 1 > /proc/sys/vm/drop_caches
```

**释放slab objects (includes dentries and inodes)**
```bash
root@localhost ~# echo 2 > /proc/sys/vm/drop_caches
```

**释放pagecache和slab objects**
```bash
root@localhost ~# echo 3 > /proc/sys/vm/drop_caches
```
使用该文件释放缓存会引起性能问题，因为内核又会花费大量的CPU时间和io去重新建立刚才清理的`pagecache`或者`slab objects`，因此，只建议在**测试和debug环境**中使用该文件。

当操作该文件时，默认情况下，在内核日志中会有如下打印信息，用于提示是哪个进程在drop caches：

```bash
root@localhost ~# dmesg 
[84448.545991] bash (23129): drop_caches: 1
[84480.242218] bash (23129): drop_caches: 3
```
这些只是提示信息，我们可以通过`echo 4 > /proc/sys/vm/drop_caches`来禁止输出该信息。

另外，内核中会为`drop_caches`操作进行计数，我们可以通过文件`/proc/vmstat`进行查看：

```bash
root@localhost ~# cat /proc/vmstat  | grep drop
drop_pagecache 4
drop_slab 3
root@localhost ~# echo 3 > /proc/sys/vm/drop_caches 
root@localhost ~# cat /proc/vmstat  | grep drop
drop_pagecache 5
drop_slab 4
```


## extfrag_threshold
## hugepages_treat_as_movable
## hugetlb_shm_group
## laptop_mode
## legacy_va_layout
## lowmem_reserve_ratio
## max_map_count
## memory_failure_early_kill
## memory_failure_recovery
## min_free_kbytes
## min_slab_ratio
## min_unmapped_ratio
## mmap_min_addr
## mmap_rnd_bits
## mmap_rnd_compat_bits
## nr_hugepages
## nr_hugepages_mempolicy
## nr_overcommit_hugepages
## nr_pdflush_threads
## numa_zonelist_order
## oom_dump_tasks
## oom_kill_allocating_task
## overcommit_kbytes
## overcommit_memory
## overcommit_ratio
## page-cluster
## panic_on_oom
## percpu_pagelist_fraction
## stat_interval
## swappiness
## user_reserve_kbytes
## vfs_cache_pressure
## zone_reclaim_mode

