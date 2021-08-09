---
layout:     post
title:      linux cgroup
subtitle:   linux cgroup 一探究竟
date:       2021-08-09
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    -  docker
---

# cgroup是什么？

cgroup又可以称为控制组，主要做资源隔离。**其原理是将一组进程放在一个控制组里，通过给这个控制组分配指定的可用资源**，达到控制这一组进程可用资源的目的。

它可以限制被namespace隔离起来的资源，还可以为资源设置权重，计算使用量，控制任务(进程或线程)启停等。

> cgroups是Linux内核提供的一种机制，这种机制可以根据需求把一系列任务(进程或线程)及其子任务整合(或分隔)到按资源划分等级的不同组内。

- 通俗的说：cgroups可以限制，记录进程组所使用的物理资源(CPU,Memory,IO等)。
- 本质上来说：cgroups是内核附加在程序上的一系列钩子，通过程序运行时对资源的调度触发相应的钩子，以达到资源追踪和限制的目的。

Cgroup可以对进程进行任意分组，并对这一组进程进行以上资源的限制，即把任务放在一个组里统一控制。

> 在Linux系统中，内核本身的调度和管理并不对进程和线程进行区分，只根据clone创建时传入参数的不同，来从概念上区别进程和线程，所以本节统一称之为任务。

对于开发者来说，cgroups有如下4个特点：

- cgroups的API以一个伪文件系统的方式实现，用户态的程序可以通过文件操作实现cgroups的组织管理。
- cgroups的组织管理操作单元可以细粒度到线程级别，另外用户可以创建和销毁cgroup，从而实现资源再分配和管理。
- 所有资源管理的功能都以子系统的方式实现，接口统一。
- 子任务创建之初与其父任务处于同一个cgroups的控制组。

# cgroup的作用

实现cgroups的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。

其主要提供以下4个功能：

1. 资源限制:cgroups可以对任务使用的资源总额进行限制。如设定应用运行时使用内存上限，一旦超过这个配额就发出OOM提示。
2. 优先级分配:通过分配CPU时间片数量及磁盘IO带宽大小，实际上就相当于控制了任务运行的优先级。
3. 资源统计：cgroups可以统计系统资源的使用量，如CPU使用时长，内存用量等。这个功能适合用于计费。
4. 任务控制：cgroups可以对任务执行挂起，恢复等操作。

# 资源限制子系统

从实现的角度来看，Cgroup实现了一个通用的进程分组框架，而不同资源的具体管理则是由各个Cgroup子系统实现的:

- blkio：可以为块设备设定输入/输出限制，比如物理驱动设备（包括磁盘、固态硬盘、USB等）。
- cpu：使用调度程序控制任务对CPU的使用。
- cpuacct：自动生成cgroup中任务对CPU资源使用情况的报告。
- cpuset：可以为cgroup中的任务分配独立的CPU（此处针对多处理器系统）和内存。
- devices：可以开启或关闭cgroup中任务对设备的访问。
- freezer：可以挂起或恢复cgroup中的任务。
- memory：可以设定cgroup中任务对内存使用量的限定，并且自动生成这些任务对内存资源使用情况的报告。
- perf_event：使用后使cgroup中的任务可以进行统一的性能测试。
- net_cls：Docker没有直接使用它，它通过使用等级识别符（classid）标记网络数据包，从而允许Linux流量控制程序（Traffic Controller，TC）识别从具体cgroup中生成的数据包。

Linux中cgroup的实现形式表现为一个文件系统，因此需要mount这个文件系统才能够使用（也有可能已经mount好了），挂载成功后，就能看到各类子系统。

### 查看CPU子系统下的控制组文件，并进行资源控制

```
## 查看CPU子系统下的控制组文件
root@iZ2ynfh5pnexr5Z:~# ls /sys/fs/cgroup/cpu
aegis                  cpuacct.stat          cpu.shares         release_agent
assist                 cpuacct.usage         cpu.stat           system.slice
cgroup.clone_children  cpuacct.usage_percpu  docker             tasks
cgroup.procs           cpu.cfs_period_us     init.scope         user.slice
cgroup.sane_behavior   cpu.cfs_quota_us      notify_on_release

## 在cpu控制组下创建新的cgroup cg1
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu# pwd
/sys/fs/cgroup/cpu
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu# ls
aegis                  cpuacct.stat          cpu.shares         release_agent
assist                 cpuacct.usage         cpu.stat           system.slice
cgroup.clone_children  cpuacct.usage_percpu  docker             tasks
cgroup.procs           cpu.cfs_period_us     init.scope         user.slice
cgroup.sane_behavior   cpu.cfs_quota_us      notify_on_release
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu# mkdir cg1

## 可以看到，创建完cg1后，cg1下便会有很多类似的文件
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu# ls ./cg1/
cgroup.clone_children  cpuacct.usage         cpu.cfs_quota_us  notify_on_release
cgroup.procs           cpuacct.usage_percpu  cpu.shares        tasks
cpuacct.stat           cpu.cfs_period_us     cpu.stat


## 对当前进程进行cpu限制
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/cg1# pwd
/sys/fs/cgroup/cpu/cg1
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/cg1# echo $$
27008
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/cg1# echo 27008 >>/sys/fs/cgroup/cpu/cg1/tasks
## cpu限制最高使用为20% 
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/cg1# echo 20000 >>/sys/fs/cgroup/cpu/cg1/cpu.cfs_quota_us
```

在docker的实现中,Docker daemon会在单独挂载了每一个子系统的控制组目录（比如/sys/fs/ cgroup/cpu）下创建一个名为docker的控制组，然后在docker控制组里面，再为每个容器创建一个以容器ID为名称的容器控制组，这个容器里的所有进程的进程号都会写到该控制组tasks中，并且在控制文件（比如cpu.cfs_quota_us）中写入预设的限制参数值。

```
## 查看docker容器，目标容器id为"8676be853e8d"
f6d9eb1ab7f0a# docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                                                                                        NAMES
8676be853e8d        zookeeper                   "/docker-entrypoint.…"   2 months ago        Up 2 months         2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp 

## 进入对应目录"/sys/fs/cgroup/cpu/docker/8676be853e8d198f967ae50e47d0f6e466c7459fba6af987a7af6d9eb1ab7f0a"
cd /sys/fs/cgroup/cpu/docker/8676be853e8d198f967ae50e47d0f6e466c7459fba6af987a7af6d9eb1ab7f0a

## ls
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/docker/8676be853e8d198f967ae50e47d0f6e466c7459fba6af987a7af6d9eb1ab7f0a# ls
cgroup.clone_children  cpuacct.usage         cpu.cfs_quota_us  notify_on_release
cgroup.procs           cpuacct.usage_percpu  cpu.shares        tasks
cpuacct.stat           cpu.cfs_period_us     cpu.stat

## cat tasks
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/docker/8676be853e8d198f967ae50e47d0f6e466c7459fba6af987a7af6d9eb1ab7f0a# cat tasks 
20299
20358
20359
20360
20361
20362
...

## cat cpu.cfs_quota_us
root@iZ2ynfh5pnexr5Z:/sys/fs/cgroup/cpu/docker/8676be853e8d198f967ae50e47d0f6e466c7459fba6af987a7af6d9eb1ab7f0a# cat cpu.cfs_quota_us
-1
```

# docker实现方式

- 工作原理

cgroups的实现本质上是给任务挂上钩子，当任务运行的过程中涉及某种资源时，就会触发钩子上所附带的子系统进行检测，根据资源类别的不同，使用对应的技术进行资源限制和优先级分配。

- 如何判断资源超出限制

对于不同的系统资源，cgroups提供了统一的接口对资源进行控制和统计，但限制的具体方式则不尽相同。

比如memory子系统，会在描述内存状态的“mm_struct”结构体中记录它所属的cgroup，当进程需要申请更多内存时，就会触发cgroup用量检测，用量超过cgroup规定的限额，则拒绝用户的内存申请，否则就给予相应内存并在cgroup的统计信息中记录。
进程所需的内存超过它所属的cgroup最大限额以后，如果设置了OOM Control（内存超限控制），那么进程就会收到OOM信号并结束；否则进程就会被挂起，进入睡眠状态，直到cgroup中其他进程释放了足够的内存资源为止。(。Docker中默认是开启OOM Control的)

- cgroup 如何与任务进行关联？

实现上，cgroup与任务之间是多对多的关系，所以它们并不直接关联，而是通过一个中间结构把双向的关联信息记录起来。(类似关联信息表记录多对多关系)

每个任务结构体task_struct中都包含了一个指针，可以查询到对应cgroup的情况，同时也可以查询到各个子系统的状态，这些子系统状态中也包含了找到任务的指针(互相持有指针)。

- docker实际上是如何使用cgroup？

在实际的使用过程中，Docker需要通过挂载cgroup文件系统新建一个层级结构，挂载时指定要绑定的子系统。把cgroup文件系统挂载上以后，就可以像操作文件一样对cgroups的层级进行浏览和操作管理（包括权限管理、子文件管理等）(mount)。

-  /sys/fs/cgroup/cpu/docker/<container-ID>下文件的作用

以资源开头（比如cpu.shares）的文件都是用来限制这个cgroup下任务的可用的配置文件。

一个cgroup创建完成，不管绑定了何种子系统，其目录下都会生成以下几个文件，用来描述cgroup的相应信息。同样，把相应信息写入这些配置文件就可以生效。主要内容如下：

- tasks：这个文件中罗列了所有在该cgroup中任务的TID，即所有进程或线程的ID。把一个任务的TID写到这个文件中就意味着把这个任务加入这个cgroup中，如果这个任务所在的任务组(本身是线程，则这个线程所属进程ID)与其不在同一个cgroup，那么会在cgroup.procs文件里记录一个该任务所在任务组的TGID值，但是该任务组的其他任务并不受影响。
- cgroup.procs：这个文件罗列所有在该cgroup中的TGID（线程组ID），即线程组中第一个进程的PID。该文件并不保证TGID有序和无重复。写一个TGID到这个文件就意味着把与其相关的线程都加到这个cgroup中。
- notify_on_release：填0或1，表示是否在cgroup中最后一个任务退出时通知运行release agent，默认情况下是0，表示不运行。
- release_agent：指定release agent执行脚本的文件路径（该文件在最顶层cgroup目录中存在），这个脚本通常用于自动化卸载无用的cgroup。

