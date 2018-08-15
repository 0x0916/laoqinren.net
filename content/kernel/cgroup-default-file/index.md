---
title: "Cgroup 中默认文件的内核实现分析"
date: 2018-08-15T10:09:56+08:00
lastmod: 2018-08-15T10:09:56+08:00
draft: false 
keywords: []
description: ""
tags: ["kernel", "linux", "cgroup"]
categories: ["kernel", "cgroup"]
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

cgroup中有一些默认的文件，本文详细分析了这些文件背后的内核实现细节，以便更深入的理解cgroup。

这些文件包括：

* notify_on_release
* release_agent
* cgroup.procs
* tasks
* cgroup.clone_children
* cgroup.event_control
* cgroup.sane_behavior

>  注意：本文分析中引用的代码来自于centos系统自带的内核：[kernel-3.10.0-862.9.1.el7](http://vault.centos.org/7.5.1804/updates/Source/SPackages/kernel-3.10.0-862.9.1.el7.src.rpm)

<!--more-->

### notify_on_release

`notify_on_release`接口在每个cgroup都存在，对其读写本质上是修改该`cgroup->flags`的`CGRP_NOTIFY_ON_RELEASE`, 后续cgroup释放时，会根据该flags进行相应的操作。

```c
static struct cftype files[] = {
	... 
	... 
	{
                .name = "notify_on_release",
                .read_u64 = cgroup_read_notify_on_release,
                .write_u64 = cgroup_write_notify_on_release,
        },
	... 
	... 
	{}
}

static u64 cgroup_read_notify_on_release(struct cgroup *cgrp,
                                            struct cftype *cft)
{
        return notify_on_release(cgrp);
}

static int cgroup_write_notify_on_release(struct cgroup *cgrp,
                                          struct cftype *cft,
                                          u64 val)
{
        clear_bit(CGRP_RELEASABLE, &cgrp->flags);
        if (val)
                set_bit(CGRP_NOTIFY_ON_RELEASE, &cgrp->flags);
        else
                clear_bit(CGRP_NOTIFY_ON_RELEASE, &cgrp->flags);
        return 0;
}

```

从`cgroup_write_notify_on_release`可以看出:

* 写入0或者空都可以清除CGRP_NOTIFY_ON_RELEASE这个flags
* 写入1或者任意正整数，都可以置位CGRP_NOTIFY_ON_RELEASE这个flags

示例如下：

```bash
~  # # 进入memory cgroup，准备实验
~  # cd /sys/fs/cgroup/memory/
/sys/fs/cgroup/memory  # # 创建测试用的cgroup
/sys/fs/cgroup/memory  # mkdir test1 test2
/sys/fs/cgroup/memory  # cat test1/notify_on_release 
0
/sys/fs/cgroup/memory  # cat test2/notify_on_release 
0
/sys/fs/cgroup/memory  # # 写入0或者一个正数
/sys/fs/cgroup/memory  # echo 1 > test1/notify_on_release 
/sys/fs/cgroup/memory  # echo 40 > test2/notify_on_release 
/sys/fs/cgroup/memory  # # 查看结果，notify_on_release都置位为1
/sys/fs/cgroup/memory  # cat test1/notify_on_release 
1
/sys/fs/cgroup/memory  # cat test2/notify_on_release 
1
/sys/fs/cgroup/memory  # # 写入0或者空清除该flags
/sys/fs/cgroup/memory  # echo > test1/notify_on_release 
/sys/fs/cgroup/memory  # echo 0 > test2/notify_on_release 
/sys/fs/cgroup/memory  # # 查看结果，notify_on_release都置位为0
/sys/fs/cgroup/memory  # cat test1/notify_on_release 
0
/sys/fs/cgroup/memory  # cat test2/notify_on_release 
0
/sys/fs/cgroup/memory  # # 清理
/sys/fs/cgroup/memory  # rmdir test1 test2
```

### release_agent

release_agent里面包含了cgroup退出时将会执行的命令，系统调用该命令时会将相应cgroup的相对路径当作参数传进去。

> 注意：这个文件只会存在于root cgroup下面，其他cgroup里面不会有这个文件。

当cgroup退出是，如果notify_on_release为1，则会调用release_agent里面配置的命令。

内核中，该文件对应的就是一个字符串，其保存在`cgroup->root->release_agent_path`中。 对该文件的读写就不分析了，这里主要分析一下内核中如何实现`cgroup`退出时，执行`release_agent`中指定的命令。

```
/* the list of cgroups eligible for automatic release. Protected by
 * release_list_lock */
static LIST_HEAD(release_list);
static DEFINE_RAW_SPINLOCK(release_list_lock);
static void cgroup_release_agent(struct work_struct *work);
static DECLARE_WORK(release_agent_work, cgroup_release_agent);
static void check_for_release(struct cgroup *cgrp);
```

* 在`cgroup_destroy_locked`和`__put_css_set`等`cgroup`退出的代码路径中，都会调用`check_for_release`函数，该函数用来判断是否需要执行`release_agent`指定的命令，如果需要，就将该cgroup添加到链表`release_list`中,并调度`schedule_work(&release_agent_work);`。
* `cgroup_release_agent`这个方法，从`release_list`链表中循环读取要退出的cgroup，然后构建执行环境，去执行指定的命令。

```c
static void cgroup_release_agent(struct work_struct *work)
{
        BUG_ON(work != &release_agent_work);
        mutex_lock(&cgroup_mutex);
        raw_spin_lock(&release_list_lock);
        while (!list_empty(&release_list)) {
                char *argv[3], *envp[3];
                int i;
                char *pathbuf = NULL, *agentbuf = NULL;
                struct cgroup *cgrp = list_entry(release_list.next,
                                                    struct cgroup,
                                                    release_list);
                list_del_init(&cgrp->release_list);
                raw_spin_unlock(&release_list_lock);
                pathbuf = kmalloc(PAGE_SIZE, GFP_KERNEL);
                if (!pathbuf)
                        goto continue_free;
                if (cgroup_path(cgrp, pathbuf, PAGE_SIZE) < 0)
                        goto continue_free;
                agentbuf = kstrdup(cgrp->root->release_agent_path, GFP_KERNEL);
                if (!agentbuf)
                        goto continue_free;

                i = 0;
                argv[i++] = agentbuf;
                argv[i++] = pathbuf;
                argv[i] = NULL;

                i = 0;
                /* minimal command environment */
                envp[i++] = "HOME=/";
                envp[i++] = "PATH=/sbin:/bin:/usr/sbin:/usr/bin";
                envp[i] = NULL;

                /* Drop the lock while we invoke the usermode helper,
                 * since the exec could involve hitting disk and hence
                 * be a slow process */
                mutex_unlock(&cgroup_mutex);
                call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
                mutex_lock(&cgroup_mutex);
 continue_free:
                kfree(pathbuf);
                kfree(agentbuf);
                raw_spin_lock(&release_list_lock);
        }
        raw_spin_unlock(&release_list_lock);
        mutex_unlock(&cgroup_mutex);
}
```

可以看出，`release_agent` 中指定的命令的执行时，其`HOME`目录为`/`。另外，这种设计可以让`cgroup`退出和执行`release_agent` 中指定的命令异步进行。


### cgroup.procs
### tasks
### cgroup.clone_children
### cgroup.event_control
### cgroup.sane_behavior

### 参考文章

