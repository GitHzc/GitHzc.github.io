---
title: Linux的mount namespace
tags:
- Linux
categories: Linux
---

## mount namespace介绍
  Linux的namespace技术可以用资源隔离，起到虚拟化的效果，使用namespace和cgroup就能实现类似于docker的简单容器。namespace有以下类别，对不同的资源起到隔离效果：
<!-- more -->
名称               | 作用
------------------ | -----------------------------------------------
mount namespace    | 隔离文件系统
UTS namespace      | 隔离主机名和域名
IPC namespace      | 隔离用于进程间通信的消息队列、信号量、共享内存
network namespace  | 隔离ipv4、ipv6栈，路由表，防火墙规则等资源
PID namespace      | 隔离进程pid
user namespace     | 隔离uid，gid
这里主要介绍mount namespace这个命名空间。

- mount namespace隔离的是文件系统，不同mount namespace中的进程看到的mount信息不同，目录树也就不同。
- /proc/[pid]/mounts，/proc/[pid]/mountinfo，/proc/[pid]/mountstats这几个文件提供了对应的mount namespace的视图。
- 当进程调用clone或unshare，并且使用了CLONE_NEWNS这一flag时，就会创建出新的mount namespace，新的mount namespace的挂载点列表（mount point list）是调用进程的挂载点列表的一个副本。
- 对挂载点列表的操作不会影响到其他mount namespace中的挂载点列表，就是在一个mount namespace中mount或者umount，别的mount namespace不会感知到这种变化。

但是有时我们并不想要如此强的隔离性，比如想挂载一个device，那么就得在每个mount namespace中都mount一次，很不方便。所幸Linux还提供了一种shared subtrees的机制，能使mount或umount的事件在不同的namespace中传播。

## shared subtrees机制
shared subtrees特性在Linux 2.6.15的版本引入，缓解了mount namespace的隔离性所带来的问题。这个特性允许mount和umount事件发生时，能够从属于同一peer group的namespace传播到另一个namespace，使得一次mount或umout操作能对多个namespace中的挂载点列表起作用。

每个挂载点有一个传播类型（propagation type），决定了在这个挂载点上发生的mount或umount事件怎样传播。

挂载点类型   | 说明
-------------|------------------------------------------------------------
MS_SHARED    |此类型的挂载点将事件传播到peer group中的其他挂载点，同时接收来自peer group的其他挂载点下发生的事件。
MS_PRIVATE   |此类型的挂载点是private的，不参加peer group，事件既不会传进来，也不会传出去。
MS_SLAVE     |事件从指定为master的peer group中传进来，此挂载点中发生的事件不会传出去。
MS_UNBINDABLE|类似与MS_PRIVATE，此外，这种挂载点不能使用mount命令的bind功能。

**注意：**

- 一个挂载点可以是一个peer group的slave的同时，能够与另一个peer group分享事件。
- 挂载点类型只决定挂载点下发生的mount或umount事件会不会传播，不影响挂载点下的挂载点内发生的事件。比如挂载点"/"的类型只影响"/"下发生的事件，不影响挂载点"/home"下发生的事件。
- 由上一条可知，一个挂载点被umount的事件会不会传播，取决与父挂载点的类型。

类型为MS_SHARED的挂载点加入到peer group中的两种方式：

1. 创建新mount namespace时，挂载点被复制。
2. 从挂载点bind mount。

在这两种情况下，新的挂载点加入到原来挂载点所在的peer group中。

peer group的创建：在已有的**类型为MS_SHARED**的挂载点下创建子挂载点，在这种情况下子挂载点被标记为MS_SHARED，得到的peer group包含在父挂载点下复制的挂载点，以及在父挂载点的peer挂载点下复制的挂载点。

要查看mount namespace中挂载点的类型，可以通过/proc/[pid]/mountinfo文件。文件中在optional字段能够看到下列标签，代表不同的挂载点类型。

- shared:X
- master:X
- propagate_from:X
- unbindable

X代表peer group的ID，有内核自动产生且是唯一的。需要注意的是propagate_from:X这个标签，它表示挂载点是一个slave，总是与master:X一起出现，这里的X表示进程的根目录下最近的占优势的peer group。如果挂载点的直接master是X，或者没有占优势的peer group，就只有master:X字段会显示出来，而不显示propagate_from:X字段。如果没有上述的标签，那这个挂载点就是private类型。

## 挂载点类型示例
#### MS_SHARED和MS_PRIVATE
以下命令在root下执行
```
1、创建挂载点/mntS，/mntP
mount /dev/sda1 /mntS
mount /dev/sda2 /mntS
2、设置/mntS的类型为MS_SHARED，/mntP的类型为MS_PRIVATE
mount --make-shared /mntS
mount --make-private /mntP
3、查看/proc/self/mountinfo文件的内容
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

146 29 8:1 / /mntS rw,relatime shared:102
341 29 8:2 / /mntP rw,relatime
```
从命令的输出可以看到，/mntS在peer group 102中是MS_SHARED类型。/mntP没有任何标签，因此是MS_PRIVATE类型。

```
1、使用unshare命令创建新的mount namespace
unshare -m --propagation unchanged sh
2、查看/proc/self/mountinfo文件的内容
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

419 347 8:1 / /mntS rw,relatime shared:102
420 347 8:2 / /mntP rw,relatime
```
从命令的输出可以看到，新namespace复制了原来namespace的挂载点。这些复制的新挂载点保持原来的类型。（--propagation unchanged防止unshared在创建新mount namespace时将所有挂载点设置为private，这是默认的类型。）

在新的mount namespace里面执行以下命令
```
mkdir /mntS/a
mount /dev/sdb3 /mntS/a
mkdir /mntP/b
mount /dev/sdb5 /mntP/b
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

434 369 8:1 / /mntS rw,relatime shared:102
436 369 8:2 / /mntP rw,relatime
435 434 8:19 / /mntS/a rw,relatime shared:87
446 436 8:21 / /mntP/b rw,relatime
```
由结果可以看到/mntS/a和/mntP/b分别继承了父挂载点的类型。

回到原来的mount namespace，执行命令
```
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

364 140 8:1 / /mntS rw,relatime shared:102
365 140 8:2 / /mntP rw,relatime
441 364 8:19 / /mntS/a rw,relatime shared:87
```
由结果可以看到类型为MS_SHARED的挂载点/mntS下发生的mount事件传播到了原来的命名空间，但类型为MS_PRIVATE的挂载点/mntP下发生的mount事件则没有向外传播。

### MS_SLAVE
MS_SLAVE只从master shared peer group接收事件，不对外传播事件。

```
1、创建MS_SHARED类型的挂载点/mntX，/mntY
mkdir /mntX /mntY
mount /dev/sda1 /mntX
mount /dev/sda2 /mntY
mount --make-shared /mntX
mount --make-shared /mntY
2、创建新的mount namespace
unshare -m --propagation unchanged bash
3、在新的mount namespace中指定/mntY为MS_SLAVE
mount --make-slave /mntY
4、在新的mount namespace中查看/proc/self/mountinfo文件信息
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'
477 439 8:1 / /mntX rw,relatime shared:87
478 439 8:2 / /mntY rw,relatime master:92

```
在新的mount namespace中执行以下命令
```
mkdir /mntX/a
mount /dev/sdb3 /mntX/a
mkdir /mntY/b
mount /dev/sdb5 /mntY/b
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

477 439 8:1 / /mntX rw,relatime shared:87
478 439 8:2 / /mntY rw,relatime master:92
479 477 8:19 / /mntX/a rw,relatime shared:97
488 478 8:21 / /mntY/b rw,relatime
```
由结果可以看到挂载点/mntX/a继承了父挂载点的挂载类型，/mntY/b的类型则是MS_PRIVATE。

回到原来的mount namespace，执行命令
```
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

146 140 8:1 / /mntX rw,relatime shared:87
344 140 8:2 / /mntY rw,relatime shared:92
482 146 8:19 / /mntX/a rw,relatime shared:97
```
可以看到，类型为MS_SLAVE的挂载点/mntY下发生的事件并没有对外传播。

在原来的mount namespace中创建挂载点/mntY/c
```
mkdir /mntY/c
mount /dev/sdb1 /mntY/c
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

146 140 8:1 / /mntX rw,relatime shared:87
344 140 8:2 / /mntY rw,relatime shared:92
482 146 8:19 / /mntX/a rw,relatime shared:97
489 344 8:17 / /mntY/c rw,relatime shared:102
```
在新的mount namespace中执行命令
```
cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'

477 439 8:1 / /mntX rw,relatime shared:87
478 439 8:2 / /mntY rw,relatime master:92
479 477 8:19 / /mntX/a rw,relatime shared:97
488 478 8:21 / /mntY/b rw,relatime
493 478 8:17 / /mntY/c rw,relatime master:102
```
可以看到在新的mount namespace中类型为MS_SLAVE的/mntY接收到了在原来的mount namespace中类型为MS_SHARED的挂载点/mntY下发生的事件。


### MS_UNBINDABLE
MS_UNBINDABLE类型的用途之一是避免挂载点爆炸的问题。mount bind能把一个目录挂载到另一个目录，如果原来的目录对应的挂载点有多个子挂载点，那么这些子挂载点同样会被挂载到新的目录下。举个例子，原来有/, /mntX两个挂载点，把/用挂到/home/test之后就会有/，/mntX，/home/test，/home/test/mntX，然后再把/挂到/home/test2后，就会有/，/mntX，/home/test，/home/test/mntX，/home/test2，/home/test2/mntX，/home/test2/home/test，/home/test2/home/test/mntX。挂载点数目增长得非常快。

类型为MS_UNBINDABLE的挂载点则在父挂载点被mount bind时不会被复制，截断了子挂载点数目的增长。如果/mntX和/home/test类型为MS_UNBINDABLE，则第一次mount bind的结果是/，/mntX，/home/test，第二次mount bind的结果是/，/mnt，/home/test，/home/test2。有效减少挂载点的数目。

**参见：[mount namespace文档](http://man7.org/linux/man-pages/man7/mount_namespaces.7.html)**
