---
title: "Cgroup之不绑定任何subsystem"
date: 2018-08-14T19:59:57+08:00
lastmod: 2018-08-14T19:59:57+08:00
draft: false
keywords: ["cgroup"]
description: ""
tags: ["cgroup"]
categories: ["cgroup"]
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

本文将创建并挂载一颗不和任何`subsystem`系统绑定的`cgroup`树。用来演示怎么创建、删除子cgroup，以及如何往cgroup中添加和删除进程，并详细介绍了每个`cgroup`中默认都有的几个文件的含义。

最后，介绍了这样使用`cgroup`的一个用户。

> 本文所有例子在`centos 7.5` 下执行通过

<!--more-->

### 挂载cgroup树

开始使用`cgroup`前需要先挂载`cgroup`树，下面先看看如何挂载一颗`cgroup`树，然后再查看其根目录下生成的文件

```bash
# 准备需要的目录
~  # mkdir cgroup && cd cgroup
~/cgroup  # mkdir demo
# 由于name=demo的cgroup树不存在，所以系统会创建一颗新的cgroup树，然后挂载到demo目录
~/cgroup  # mount -t cgroup -o none,name=demo demo ./demo
# 挂载点所在目录就是这颗cgroup树的root cgroup，在root cgroup下面，系统生成了一些默认文件
~/cgroup  # ls ./demo/
cgroup.clone_children  cgroup.event_control  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks
# cgroup.procs里包含系统中的所有进程
~/cgroup  # wc -l ./demo/cgroup.procs 
182 ./demo/cgroup.procs
```

这些默认的文件的含义如下:

|文件名称 | 含义 |
|---|---|
|cgroup.clone_children|这个文件只对cpuset（subsystem）有影响，当该文件的内容为1时，新创建的cgroup将会继承父cgroup的配置，即从父cgroup里面拷贝配置文件来初始化新cgroup|
|cgroup.procs|当前cgroup中的所有进程ID，系统不保证ID是顺序排列的，且ID有可能重复|
|tasks|当前cgroup中的所有线程ID，系统不保证ID是顺序排列的|
|cgroup.sane_behavior||
|cgroup.event_control||
|notify_on_release|该文件的内容为1时，当cgroup退出时（不再包含任何进程和子cgroup），将调用release_agent里面配置的命令。新cgroup被创建时将默认继承父cgroup的这项配置。|
|release_agent|里面包含了cgroup退出时将会执行的命令，系统调用该命令时会将相应cgroup的相对路径当作参数传进去。 注意：这个文件只会存在于root cgroup下面，其他cgroup里面不会有这个文件。|

### 创建和删除cgroup

挂载好cgroup树之后，就可以在里面新建cgroup了，其实新建cgroup很简单，就是创建目录就可以了。

```bash
~/cgroup  # # 创建子cgroup很简单，新建一个目录就可以了
~/cgroup  # cd demo/
~/cgroup/demo  # mkdir cgroup1
~/cgroup/demo  # # 在新建的子cgroup中，系统默认也创建了一些文件，这些文件的意义和root cgroup中的一样
~/cgroup/demo  # ls cgroup1/
cgroup.clone_children  cgroup.event_control  cgroup.procs  notify_on_release  tasks
~/cgroup/demo  # # 新创建的cgroup中没有任何进程
~/cgroup/demo  # wc -l cgroup1/cgroup.procs 
0 cgroup1/cgroup.procs
~/cgroup/demo  # wc -l cgroup1/tasks 
0 cgroup1/tasks
~/cgroup/demo  # # 每个cgroup都可以创建自己的子cgroup
~/cgroup/demo  # mkdir cgroup1/cgroup11
~/cgroup/demo  # ls cgroup1/cgroup11/
cgroup.clone_children  cgroup.event_control  cgroup.procs  notify_on_release  tasks
~/cgroup/demo  # # 删除cgroup也很简单，直接删除相应的目录就可以了
~/cgroup/demo  # # 先删除cgroup11，再删除cgroup1
~/cgroup/demo  # rmdir cgroup1/cgroup11
~/cgroup/demo  # rmdir cgroup1
```

### 添加进程到cgroup

创建新的cgroup后，就可以往里面添加进程了。注意以下几点：

* 再一颗cgroup树里面，一个进程必须且只能属于一个cgroup。
* 新创建的子进程会自动添加到父进程所在的cgroup中。
* 从一个cgroup移动一个进程到另外一个cgroup中，只要有目的cgroup的写入权限就可以了，系统不会检查源cgroup里的权限。
* 用户只能操作属于自己的进程，不能操作其他用户的进程，root账号除外。

```bash
# 第一个shell窗口
~/cgroup/demo  # mkdir test
~/cgroup/demo  # cd test/
~/cgroup/demo/test  # # 将当前bash加入到上面新创建的cgroup中
~/cgroup/demo/test  # echo $$ 
13945
~/cgroup/demo/test  # echo $$ > cgroup.procs 
~/cgroup/demo/test  # # 注意：一次只能往这个文件中写一个进程ID，如果需要写多个的话，需要多次调用这个命令


# 第二个shell窗口
~/cgroup/demo/test  # # 这时可以看到cgroup.procs里面包含了上面的第一个shell进程
~/cgroup/demo/test  # cat cgroup.procs 
13945


# 第一个shell窗口
~/cgroup/demo/test # # 回到第一个窗口，运行top命令
~/cgroup/demo/test # top 
# 省略输出内容



# 第二个shell窗口
~/cgroup/demo/test  # # 这时再在第二个窗口查看，发现top进程自动和它的父进程（13945）属于同一个cgroup
~/cgroup/demo/test  # cat cgroup.procs 
13945
18314
~/cgroup/demo/test  # ps -ef|grep top
root     18314 13945  0 22:30 pts/2    00:00:00 top
root     18389 17939  0 22:31 pts/3    00:00:00 grep --color=auto top
~/cgroup/demo/test  # # 在一颗cgroup树里面，一个进程必须要属于一个cgroup，
~/cgroup/demo/test  # # 所以我们不能凭空从一个cgroup里面删除一个进程，只能将一个进程从一个cgroup移到另一个cgroup
~/cgroup/demo/test  # # 这里我们将13945移动到root cgroup
~/cgroup/demo/test  # echo 13945 > ../cgroup.procs 
~/cgroup/demo/test  # cat cgroup.procs 
18314
~/cgroup/demo/test  # # 移动13945到另外一个cgroup后，它的子进程不会随着移动


# 第一个shell窗口
~/cgroup/demo/test  # #回到第一个shell窗口，进行清理工作
~/cgroup/demo/test  # # 先用ctrl+c退出top命令
~/cgroup/demo/test  # cd ..
~/cgroup/demo  # # 然后删除创建的cgroup
~/cgroup/demo  # rmdir test

```

### procs 和tasks 的区别

上面提到cgroup.procs包含的是进程ID， 而tasks里面包含的是线程ID，那么他们有什么区别呢？

```bash
~/cgroup/demo  # # 创建两个新的cgroup用于演示
~/cgroup/demo  # mkdir c1 c2
~/cgroup/demo  # # 系统中找一个有多个线程的进程
~/cgroup/demo  # ps -efL | grep NetworkManager                                                                                                                
root       859     1   859  0    3 Aug12 ?        00:00:01 /usr/sbin/NetworkManager --no-daemon
root       859     1   868  0    3 Aug12 ?        00:00:00 /usr/sbin/NetworkManager --no-daemon
root       859     1   872  0    3 Aug12 ?        00:00:00 /usr/sbin/NetworkManager --no-daemon
root     15850   859 15850  0    1 21:59 ?        00:00:00 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/dhclient-enp0s3.pid -lf /var/lib/NetworkManager/dhclient-302ccf17-6708-45d0-ae97-afb55838c34b-enp0s3.lease -cf /var/lib/NetworkManager/dhclient-enp0s3.conf enp0s3
root     19169 13945 19169  0    1 22:41 pts/2    00:00:00 grep --color=auto NetworkManager
~/cgroup/demo  # # 进程859 有三个线程，分别为859 868 872
~/cgroup/demo  # 
~/cgroup/demo  # # 将868 加入到c1/cgroup.procs
~/cgroup/demo  # echo 868 > c1/cgroup.procs 
~/cgroup/demo  # # 由于cgroup.procs存放的是进程ID，所以这里看到的是868所属的进程ID（859）                                                                     
~/cgroup/demo  # cat c1/cgroup.procs 
859
~/cgroup/demo  # # #从tasks中的内容可以看出，虽然只往cgroup.procs中加了线程868
~/cgroup/demo  # # 但系统已经将这个线程所属的进程的所有线程都加入到了tasks中，
~/cgroup/demo  # # 说明现在整个进程的所有线程已经处于c1中了
~/cgroup/demo  # cat c1/tasks 
859
868
872
~/cgroup/demo  # # 将868 加入到c2/tasks中                                                                                                                     
~/cgroup/demo  # echo 868 > c2/tasks 
~/cgroup/demo  # # 这时我们看到虽然在c1/cgroup.procs和c2/cgroup.procs里面都有859
~/cgroup/demo  # cat c1/cgroup.procs 
859
~/cgroup/demo  # cat c2/cgroup.procs 
859
~/cgroup/demo  # # 但c1/tasks和c2/tasks中包含了不同的线程，说明这个进程的两个线程分别属于不同的cgroup
~/cgroup/demo  # cat c1/tasks 
859
872
~/cgroup/demo  # cat c2/tasks 
868
~/cgroup/demo  # # 通过tasks，我们可以实现线程级别的管理，但通常情况下不会这么用
~/cgroup/demo  # # 并且在cgroup V2以后，将不再支持该功能，只能以进程为单位来配置cgroup
~/cgroup/demo  # # 清理
~/cgroup/demo  # echo 859 > ./cgroup.procs 
~/cgroup/demo  # rmdir c1
~/cgroup/demo  # rmdir c2
```

### release_agent

当一个cgroup里没有进程也没有子cgroup时，release_agent将被调用来执行cgroup的清理工作。


```bash
~/cgroup/demo  # # 创建新的cgroup用于演示
~/cgroup/demo  # mkdir test
~/cgroup/demo  # # 先enable release_agent
~/cgroup/demo  # echo 1 > ./test/notify_on_release 
~/cgroup/demo  # 
~/cgroup/demo  # # 创建一个脚本 ~/release_demo.sh
~/cgroup/demo  # cat > ~/release_demo.sh <<EOF
> #!/bin/bash
> echo \$0:\$1 >> ~/release_demo.log
> EOF
~/cgroup/demo  # # 一般情况下都会利用这个脚本执行一些cgroup的清理工作，但我们这里为了演示简单，仅仅只写了一条日志到指定文件
~/cgroup/demo  # 
~/cgroup/demo  # # 添加可执行权限
~/cgroup/demo  # chmod a+x ~/release_demo.sh 
~/cgroup/demo  # # 将该脚本设置进文件release_agent
~/cgroup/demo  # echo /root/release_demo.sh > ./release_agent 
~/cgroup/demo  # cat release_agent 
/root/release_demo.sh
~/cgroup/demo  # # 往test里面添加一个进程，然后再移除，这样就会触发release_demo.sh
~/cgroup/demo  # echo $$
13945
~/cgroup/demo  # echo $$ > ./test/cgroup.procs 
~/cgroup/demo  # echo $$ > ./cgroup.procs 
~/cgroup/demo  # # 从日志可以看出，release_agent被触发了，/test是cgroup的相对路径
~/cgroup/demo  # cat /release_demo.log 
/root/release_demo.sh:/test
#  结合log中的信息和文件的路径，我们发现内核执行release_agent时，其HOME目录为/

```

### cgroup在systemd 中的应用

一般情况下，对于没有和任何subsystem关联的cgroup，在上面的所有操作没有实际意义，但是systemd软件的确使用了cgroup的这种用法。

在使用systemd的发行版上，一般都会有如下目录`/sys/fs/cgroup/systemd/`：

```bash
~/cgroup/demo  # ls -l /sys/fs/cgroup/systemd/
total 0
-rw-r--r--   1 root root 0 Aug  8 02:36 cgroup.clone_children
--w--w--w-   1 root root 0 Aug  8 02:36 cgroup.event_control
-rw-r--r--   1 root root 0 Aug  8 02:36 cgroup.procs
-r--r--r--   1 root root 0 Aug  8 02:36 cgroup.sane_behavior
-rw-r--r--   1 root root 0 Aug  8 02:36 notify_on_release
-rw-r--r--   1 root root 0 Aug  8 02:36 release_agent
drwxr-xr-x 109 root root 0 Aug 14 22:05 system.slice
-rw-r--r--   1 root root 0 Aug  8 02:36 tasks
drwxr-xr-x   4 root root 0 Aug  8 02:36 user.slice
~/cgroup/demo  # mount | grep systemd
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
```

`/sys/fs/cgroup/systemd/`目录其实就是一个没有和任何subsystem关联的cgroup，那systemd用它来做什么呢？我猜是用来追踪每个服务的进程的pid号的(由于没有找打相关文档)。

这里先将猜测记录如下，后续会进行验证。

一般使用systemctl去管理系统服务时，当我们要reload这个服务时，一般的service描述文件中记录的要执行的命令如下：

```bash
/usr/lib/systemd/system/NetworkManager.service:12:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/virtlockd.service:11:ExecReload=/bin/kill -USR1 $MAINPID
/usr/lib/systemd/system/crond.service:8:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/libvirtd.service:24:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/virtlogd.service:11:ExecReload=/bin/kill -USR1 $MAINPID
/usr/lib/systemd/system/lldpad.service:8:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/libstoragemgmt.service:7:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/auditd.service:23:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/firewalld.service:13:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/httpd.service:12:ExecStop=/bin/kill -WINCH ${MAINPID}
/usr/lib/systemd/system/smartd.service:8:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/docker.service:32:ExecReload=/bin/kill -s HUP $MAINPID
/usr/lib/systemd/system/ksmtuned.service:8:ExecReload=/bin/kill -USR1 $MAINPID
/usr/lib/systemd/system/sshd.service:11:ExecReload=/bin/kill -HUP $MAINPID
/usr/lib/systemd/system/anaconda-sshd.service:17:ExecReload=/bin/kill -HUP $MAINPID
```

那我们如何知道`$MAINPID`的值是多少呢？我猜这里就使用了该服务对应的cgroup中的cgroup.procs的输出结果。


下面通过代码来验证上面的猜测，如下引用的代码来自于：[https://github.com/systemd/systemd/tree/v219](https://github.com/systemd/systemd/tree/v219)

`systemd`有方法`service_search_main_pid`来获取`MAINPID`,其又调用了方法`unit_search_main_pid`。

`unit_search_main_pid` 代码如下：

```c
pid_t unit_search_main_pid(Unit *u) {
        _cleanup_fclose_ FILE *f = NULL;
        pid_t pid = 0, npid, mypid;

        assert(u);

        if (!u->cgroup_path)
                return 0;

        if (cg_enumerate_processes(SYSTEMD_CGROUP_CONTROLLER, u->cgroup_path, &f) < 0)
                return 0;

        mypid = getpid();
        while (cg_read_pid(f, &npid) > 0)  {
                pid_t ppid;

                if (npid == pid)
                        continue;

                /* Ignore processes that aren't our kids */
                if (get_parent_of_pid(npid, &ppid) >= 0 && ppid != mypid)
                        continue;

                if (pid != 0) {
                        /* Dang, there's more than one daemonized PID
                        in this group, so we don't know what process
                        is the main process. */
                        pid = 0;
                        break;
                }

                pid = npid;
        }

        return pid;
}
```

这个函数就非常简单了，第10行用于获取`cgroup.procs`的文件描述符，14行循环读取该文件中的`pid`。一些关键的宏和函数定义如下：

```c
#define SYSTEMD_CGROUP_CONTROLLER "name=systemd"

// 打开文件的函数
int cg_enumerate_processes(const char *controller, const char *path, FILE **_f) {
        _cleanup_free_ char *fs = NULL;
        FILE *f;
        int r;

        assert(_f);

        r = cg_get_path(controller, path, "cgroup.procs", &fs);
        if (r < 0) 
                return r;

        f = fopen(fs, "re");
        if (!f) 
                return -errno;

        *_f = f; 
        return 0;
}
```


### 参考文章

* https://segmentfault.com/a/1190000007241437
* https://github.com/systemd/systemd/tree/v219
