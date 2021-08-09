---
layout:     post
title:      linux 常用命令
subtitle:   linux 常用命令使用一览
date:       2021-08-09
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    -  linux
---

# tree

解释: 以树状展示linux文件目录

```
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# sudo apt-get install tree // 安装tree命令
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# tree .
.
├── t1
│   └── t1.log
└── t2
    └── t2.log
```

# mount --bind

- 解释:我们可以通过mount --bind命令来将两个目录连接起来，mount --bind命令是将前一个目录挂载到后一个目录上，所有对后一个目录的访问其实都是对前一个目录的访问。

- 操作:

```
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# tree .
.
├── t1
│   └── t1.log
└── t2
    └── t2.log

##  查看t1目录
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# ls -lid t1 
1453544 drwxr-xr-x 2 root root 4096 Aug  9 22:15 t1 

## 查看t1目录
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# ls -lid t2
1453545 drwxr-xr-x 2 root root 4096 Aug  9 22:15 t2

## 执行mount --bind将t1挂载到t2，可以发现inode号都变为t1的inode
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# mount --bind t1 t2
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# ls -lid t1
1453544 drwxr-xr-x 2 root root 4096 Aug  9 22:15 t1
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# ls -lid t2
1453544 drwxr-xr-x 2 root root 4096 Aug  9 22:15 t2

## 对t2的访问或修改实际上是对t1
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# tree .
.
├── t1
│   └── t1.log
└── t2
    └── t1.log
root@iZ2ynfh5pnexr5Z:/home/work/mount_test/t2# touch t3.log
root@iZ2ynfh5pnexr5Z:/home/work/mount_test/t2# cd ..
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# tree .
.
├── t1
│   ├── t1.log
│   └── t3.log
└── t2
    ├── t1.log
    └── t3.log

## 解挂载后，t1目录保持修改，t2保持不变
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# umount t2
root@iZ2ynfh5pnexr5Z:/home/work/mount_test# tree .
.
├── t1
│   ├── t1.log
│   └── t3.log
└── t2
    └── t2.log

2 directories, 3 files
```

- 原理:

mount --bind命令执行后，linux将会把被挂载目录的目录项(也就是该目录的block，其记录了下级目录的信息)屏蔽，即t2的下级目录被隐藏起来了(不是删除)。同时，
内核将挂载目录t1的目录项记录在内存的一个s_root对象里，在mount命令执行时，VFS会创建一个vfsmount对象，这个对象里包含了整个文件系统的所有mount信息，
其中，也包括本次mount信息，这个对象是一个Hash值对应表(hash值即对路径字符串的计算)，表里就有t1到t2两个目录的hash值对应关系。命令执行完后，当访问t2时，系统会告知/t2被屏蔽了，自动到内存中找VFS，通过vfsmount找到t1与t2的对应关系，从而读取到t1的inode，这样便读到了t1的目录内容。

- 注意:

1. mount --bind连接的两个目录的inode号码并不一样，只是被挂载目录的block被屏蔽掉，inode被重定向到挂载目录的inode（被挂载目录的inode和block依然没变）。
2. 2个目录的对应关系存在于内存里，一旦重启挂载关系就不存在了。

- 使用场景:

为测试某个新功能，必需修改某个系统文件。而这个文件在只读文件系统上（总不能为一个小小的测试就重刷固件吧），或者是虽然文件可写，但是自己对这个改动没有把握，不愿意直接修改。这时候mount --bind就是你的好帮手。 
假设我们要改的文件是/etc/hosts，可按下面的步骤操作： 
1. 把新的hosts文件放在/tmp下。当然也可放在硬盘或U盘上。 
2. mount --bind /tmp/hosts /etc/hosts       此时的/etc目录是可写的(挂载前不可写)，所做修改不会应用到原来的/etc目录，可以放心测试。测试完成了执行 umount /etc/hosts 断开绑定。 