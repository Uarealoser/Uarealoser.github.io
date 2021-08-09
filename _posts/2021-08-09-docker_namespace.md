---
layout:     post
title:      linux namespace
subtitle:   linux namespace 一探究竟
date:       2021-08-09
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    -  docker
---

# Linux Namespace到底是什么？

Linux 的namespace 是Linux提供的一种内核级别的环境隔离方法。通过内核对系统资源进行隔离和虚拟化。

> 官方描述:Linux将全局系统资源封装在一个抽象中，从而使namespace内的进程认为自己具有独立的资源实例。

这些系统资源包括进程ID，主机名，网络访问，进程间通讯和文件系统等。在namespace中，每一个进程都绑定在特定的命名空间之下，且只能查看和操作绑定在此命名空间的资源。

### 查看进程所属的namespace

从内核3.8开始，/proc/[pid]/ns目录下就会包含进程所属的namespace信息:

```
root@iZ2ynfh5pnexr5Z:/proc# ll /proc/$$/ns
total 0
dr-x--x--x 2 root root 0 Aug  9 21:51 ./
dr-xr-xr-x 9 root root 0 Aug  9 21:49 ../
lrwxrwxrwx 1 root root 0 Aug  9 21:51 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 net -> net:[4026531957]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug  9 21:51 uts -> uts:[4026531838]
```

说明: 这些namespace文件都是链接文件，链接文件内容的格式为xxx:[inode number],其中xxx为namespace类型，inode number则用来标识一个namespace，我们也可以把它理解为namespace的ID。
如果两个进程的某个namespace文件指向同一个链接文件，则说明其相关资源在同一个namespace中。

另外，这些链接文件存在的另一个意义是:一旦这些链接文件被打开，只要打开的文件描述符(fd)存在，那么就算该namespace下的所有进程都已经结束，这个namespace也会一直存在。

除了打开文件的方式外，我们还可以通过文件挂载的方式阻止namespace被删除,比如我们可以把uts挂载到～/uts文件:

```
root@iZ2ynfh5pnexr5Z:/proc# touch ~/uts

root@iZ2ynfh5pnexr5Z:/proc# mount --bind /proc/$$/ns/uts ~/uts

root@iZ2ynfh5pnexr5Z:/proc# stat ~/uts
  File: '/root/uts'
  Size: 0         	Blocks: 0          IO Block: 4096   regular empty file
Device: 3h/3d	Inode: 4026531838  Links: 1 // 注意Inode
Access: (0444/-r--r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-08-09 22:06:03.120123041 +0800
Modify: 2021-08-09 22:06:03.120123041 +0800
Change: 2021-08-09 22:06:03.120123041 +0800
 Birth: -

root@iZ2ynfh5pnexr5Z:/proc# ll /proc/$$/ns/uts
lrwxrwxrwx 1 root root 0 Aug  9 21:51 /proc/17880/ns/uts -> uts:[4026531838] // 这里的inode number
```

可以看到，～/uts的inode和链接文件的inode number是一样的，即它们是同一个文件。

> mount --bind t1 t2 ：将2个目录或文件链接起来，将前一个目录t1挂载到后一个目录t2上，所有对后一个目录的访问都是对前一个目录的访问。可以参考[linux 命令](https://uarealoser.cn/2021/08/09/linux_exec/)

Linux提供了如下几种Namespace:

       Namespace   变量               隔离资源
       Cgroup      CLONE_NEWCGROUP   Cgroup 根目录
       IPC         CLONE_NEWIPC      System V IPC, POSIX 消息队列等
       Network     CLONE_NEWNET      网络设备，协议栈、端口等
       Mount       CLONE_NEWNS       挂载点
       PID         CLONE_NEWPID      进程ID
       User        CLONE_NEWUSER     用户和group ID
       UTS         CLONE_NEWUTS      Hostname和NIS域名

namespace的api主要由三个系统调用和一系列的/proc文件组成。

- clone()：创建新的进程
- setns()：允许指定进程加入特定的namespace
- unshare()：将指定进程移除指定的namespace

通常，我们为了指定要操作的namespace类型，需要在系统调用的flag中通过常量CLONE_NEW*指定参数，可以通过|分隔指定多个参数。

# 三个系统调用

### clone()

```go
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

- child_func:传入子进程运行主函数
- child_stack:传入子进程使用的栈空间
- flags:表示使用哪些CLONE_*标识位
- args:用于传入用户参数

clone()与fork类似，相当于把当前进程复制一份，但clone()可以更细粒度的控制与子进程可以共享的资源(其实就是通过flags来控制)，包括虚拟内存，打开的文件描述符和信号量等。
一旦指定了CLONE_NEW*，相对应的namespace就会被创建，新创建的进程也会成为该namespace中的一员。

其实，clone()还不是最底层的系统调用，而是封装了do_fork()的内核实现。

### setns()

通过setns，加入一个存在的namespace。

```
int setns(int fd, int nstype);
```

- fd:表示要加入的 namespace 的文件描述符，可以通过打开其中一个符号链接来获取，也可以通过打开 bind mount 到其中一个链接的文件来获取。
- nstype 让调用者可以去检查 fd 指向的 namespace 类型，值可以设置为前文提到的常量 CLONE_NEW*，填 0 表示不检查。如果调用者已经明确知道自己要加入了 namespace 类型，或者不关心 namespace 类型，就可以使用该参数来自动校验。

### unshare()

```
int unshare(int flags);
```

unshare() 与 clone() 类似，但它运行在原先的进程上，不需要创建一个新进程，即：先通过指定的 flags 参数 CLONE_NEW* 创建一个新的 namespace，然后将调用者加入该 namespace。最后实现的效果其实就是将调用者从当前的 namespace 分离，然后加入一个新的 namespace。

# todo list

@todo

golang实现namespace隔离程序

# 参考

- https://lwn.net/Articles/531114/
- https://coolshell.cn/articles/17010.html
- https://zhuanlan.zhihu.com/p/159362517