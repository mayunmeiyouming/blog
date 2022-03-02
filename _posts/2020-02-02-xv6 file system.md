---
layout: post
title:  "XV6: File System"
date:   2020-02-02 14:10:01 +0800
categories: [Tech]
tag: 
  - xv6
  - OS
---

> xv6文件系统

在文件系统环境的地址空间为磁盘的`内存映射`保留了固定的3GB区域，从0x10000000（`DISKMAP`）到0xD0000000（`DISKMAP + DISKMAX`）。这意味着我们的系统将只能使用3GB大小的磁盘。磁盘访问是以扇区为单位，而操作系统是以块为单位访问磁盘。在xv6中，一个块等于8个扇区，我们将相应的虚拟地址转换为扇区号，对磁盘进行读写操作。

首先我们来看看文件系统的流程：

![]({{ '/assets/images/posts/2020-02-02-xv6 file system/01.png' | prepend: site.baseurl}})

## serve_init函数

其中`serve_init`函数完成的功能比较简单，就是初始化`opentab`数组。一共完成了两个赋值：

1. `opentab[i].o_fileid = i`;
2. `opentab[i].o_fd = (struct Fd*) va`;

其中va的初始地址为`0xD0000000`，每次赋值完成后，执行`va += PGSIZE;`，PGSIZE是4096，是一页的大小。

一个有1024个opentab，代表文件系统一次只能打开1024个文件。

## fs_init函数

首先探测磁盘1是否可用，如果可用，就设置静态变量`diskno`为1，否则设置为0。接下来执行`bc_init`函数，该函数将设置缺页异常处理函数为`bc_pgfault`，并且设置超级块`struct super`。超级块包含了文件系统的块总数和根目录`struct File s_root;`。然后设置超级块`super`指向块1，`bitmap`指向块2。

## serve函数

循环接收文件请求，将文件请求类型存放在`req`中，然后根据相应的req，调用相应的函数，进行处理，处理完后将利用内存共享，将文件共享给其它环境。下面是相应的请求类型：

### FSREQ_OPEN

这是一个打开文件的请求，将调用serve_open函数。下面是函数的流程图：

![]({{ '/assets/images/posts/2020-02-02-xv6 file system/02.png' | prepend: site.baseurl}})

首先调用函数`open_alloc`分配一个`struct OpenFile`存放在变量`o`中。如果需要创建文件，则调用函数`file_create`，该函数将分配一个`struct File`。

`file_open`获取文件的`struct File`。

`file_truncate`将释放文件的一些块，并且重新设置文件的`f_size`域。

然后将返回一个`struct Fd`

### FSREQ_READ

这是一个文件读取的请求，读取数据将存储在ipc的`readRet`域。

### FSREQ_STAT

### FSREQ_FLUSH

### FSREQ_WRITE

### FSREQ_SET_SIZE

### FSREQ_SYNC
