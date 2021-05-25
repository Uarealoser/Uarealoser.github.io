---
layout:     post
title:      kubernetes基本概念
subtitle:   容器网络
date:       2021-05-24
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - kubernetes
---

# 1. 容器网络(单机模式，Liunx容器网络实现原理：网桥模式)

Linux容器可以看见的网络栈，实际是被隔离在它自己的NetWork Namespace中的。

网络栈：网卡(NetWork Interface),回环设备(Loopback Device),路由表(Routing Table)和iptables规则。对于一个进程来说，其实就构成了它发起和响应网络请求的基本环境。

作为一个容器，它可以直接声明使用宿主机的网络栈(-net=host),即不开启NetWork Namespace。

```
 docker run –d –net=host --name nginx-host nginx
```

在这种情况下，这个容器启动后，直接监听宿主机的端口。

像这样直接使用宿主机网络栈的方式，虽然可以为容器提供良好的网络性能，但也会不可避免地引入共享网络资源的问题，比如端口冲突。**在大多数情况下，我们都希望容器进程能使用自己 Network Namespace 里的网络栈，即：拥有属于自己的 IP 地址和端口。**

那么，这个被隔离的容器进程，该如何跟其他 Network Namespace 里的容器进程进行交互呢？

举个🌰：
其实可以把每一个容器看做一台主机，它们都有一套独立的“网络栈”。如果你想要实现两台主机之间的通信，最直接的办法，就是把它们用一根网线连接起来；而如果你想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。在 Linux 中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）。它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

而为了实现上述目的，Docker 项目会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。

```
root@iZ2ynfh5pnexr5Z:~# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:fa:49:51:09  
          inet addr:172.18.0.1  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Docker 通过一个Veth Pair的设备：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。

当在宿主机上启动一个容器时，默认会将Veth Peer的一个网卡置于容器内，而另一个虚拟网卡置于宿主机上，并且插入宿主机docker0网桥。多个容器位于宿主机上的网卡都会插入宿主机docker0网桥，因此，一个宿主机上运行的多个容器默认是互通的。

## 1.1 bridge网络模型

- Veth Pairs

Veth是成对出现的两张虚拟网卡，从一端发送的数据包，总会在另一端接收到。利用Veth的特性，我们可以将一端的虚拟网卡"放入"容器内，另一端接入虚拟交换机。这样，接入同一个虚拟交换机的容器之间就实现了网络互通。

- Linux Bridge

交换机是工作在数据链路层的网络设备，它转发的是二层网络包。最简单的转发策略是将到达交换机输入端口的报文，广播到所有的输出端口。当然更好的策略是在转发过程中进行学习，记录交换机端口和MAC地址的映射关系，这样在下次转发时就能够根据报文中的MAC地址，发送到对应的输出端口。

我们可以认为Linux bridge就是虚拟交换机，连接在同一个bridge上的容器组成局域网，不同的bridge之间网络是隔离的。 docker network create [NETWORK NAME]实际上就是创建出虚拟交换机。

- iptables

容器需要能够访问外部世界，同时也可以暴露服务让外界访问，这时就要用到iptables。另外，不同bridge之间的隔离也会用到iptables。

iptables 提供了四种内置的表 raw → mangle → nat → filter，优先级从高到低：

    - raw 用于配置数据包，raw中的数据包不会被系统跟踪。不常用。
    - mangle 用于对特定数据包的修改。不常用。
    - nat: 用于网络地址转换（NAT）功能（端口映射，地址映射等）。
    - filter：一般的过滤功能，默认表。
    
## 1.2 docker bridge网络模型

![](https://tva1.sinaimg.cn/large/008i3skNly1gqupptwktqj312f0u0whi.jpg)

在 Linux 中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）。它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

Docker 项目会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。

- 1. 在宿主机上起一个ubuntu容器镜像

```
docker run -itd --name ubuntu-test ubuntu
```

- 2. 进入容器，执行查看网卡状态

```
// 进入容器
docker exec -it deb /bin/bash
// ifconfig
root@debc26e2cfd8:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.2  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:ac:12:00:02  txqueuelen 0  (Ethernet)
        RX packets 5812  bytes 18406315 (18.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4833  bytes 337413 (337.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
// route
root@debc26e2cfd8:/# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.18.0.1      0.0.0.0         UG    0      0        0 eth0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

可以看到，容器中有一张eht0的网卡，它正是Veth Pair设备在容器里的这一端。

通过route命令，查看容器的路由表，我们可以看到eth0网卡是这个容器的默认路由设备。
所有对172.18.0.0/16网段的请求，也都会被交给eth0处理。

- 3. 查看宿主机网络设备

```
// ifconfig
root@iZ2ynfh5pnexr5Z:~# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:fa:49:51:09  
          inet addr:172.18.0.1  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:4839 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5818 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:270001 (270.0 KB)  TX bytes:18406757 (18.4 MB)

eth0      Link encap:Ethernet  HWaddr 00:16:3e:08:91:37  
          inet addr:172.17.32.219  Bcast:172.17.63.255  Mask:255.255.192.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:308489 errors:0 dropped:0 overruns:0 frame:0
          TX packets:156772 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:269978184 (269.9 MB)  TX bytes:63379837 (63.3 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:49328 errors:0 dropped:0 overruns:0 frame:0
          TX packets:49328 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:3062606 (3.0 MB)  TX bytes:3062606 (3.0 MB)

vethaa5b72e Link encap:Ethernet  HWaddr 8e:f6:d5:83:10:cf  
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:4839 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5818 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:337747 (337.7 KB)  TX bytes:18406757 (18.4 MB)

// brctl show
root@iZ2ynfh5pnexr5Z:~# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242fa495109	no		vethaa5b72e
```

通过ifconfig的输出，可以看到容器对应的Veth pair设备，在宿主机上是一张虚拟网卡，为vethaa5b72e，并且通过brctl show可以看到，这张虚拟网卡被插到了docker0上。

- 4. 启动另一个容器

```
root@iZ2ynfh5pnexr5Z:~# docker run -itd --name=ubuntu-test2  ubuntu
root@iZ2ynfh5pnexr5Z:~# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242fa495109	no		veth6576975
							vethaa5b72e
```

此时，veth6576975也被插在了docker0网桥上。

此时宿主机上共运行了2个容器

```
root@iZ2ynfh5pnexr5Z:~# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0a51c771ceff        ubuntu              "/bin/bash"         5 minutes ago       Up 5 minutes                            ubuntu-test2
debc26e2cfd8        ubuntu              "/bin/bash"         About an hour ago   Up About an hour                        ubuntu-test
```

它们各自的ip为(ubuntu-test:172.18.0.2,ubuntu-test2:172.18.0.3)

我们在ubuntu-test2容器中ping ubuntu-test容器，可以ping通。

## 1.3 原理

当在ubuntu-test2中ping ubuntu-test时，我们ping的数据包通过第二条规则转发到eht0网卡上，并通过Veth Pair原理，ip包出现在宿主机的veth6576975虚拟网卡，该虚拟网卡插在docker0网桥上，因此网桥进行ARP广播，广播到docker0上其他的虚拟网卡。

这样，同样连接到docker0上的其他虚拟网卡就会收到这个ARP请求，从而将ubuntu-test容器所对应的MAC地址回复给ubuntu-test2容器。

![](https://tva1.sinaimg.cn/large/008i3skNly1gqul69fi3zj31bn0rnabw.jpg)

同样的，当一个容器试图连接到另外一个宿主机时，它发出的请求数据包，首先经过 docker0 网桥出现在宿主机上。然后根据宿主机的路由表，交给宿主机的其他网卡。

![](https://tva1.sinaimg.cn/large/008i3skNly1gqumbe1hl5j31ey0rmwgf.jpg)

所以说，当你遇到容器连不通“外网”的时候，你都应该先试试 docker0 网桥能不能 ping 通，然后查看一下跟 docker0 和 Veth Pair 设备相关的 iptables 规则是不是有异常，往往就能够找到问题的答案了。

可以参考这篇文章:[Docker单机网络模型动手实验](https://github.com/mz1999/blog/blob/master/docs/docker-network-bridge.md)

# 2. 容器跨主机网络

在 Docker 的默认配置下，一台宿主机上的 docker0 网桥，和其他宿主机上的 docker0 网桥，没有任何关联，它们互相之间也没办法连通。所以，连接在这些网桥上的容器，自然也没办法进行通信了。

## 2.1 Flannel项目

Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。目前，Flannel 支持三种后端实现，分别是：

- VXLAN
- host-gw
- UDP

### 2.1.1 UDP

假设我们有2台宿主机：

- 宿主机 Node 1 上有一个容器 container-1，它的 IP 地址是 100.96.1.2，对应的 docker0 网桥的地址是：100.96.1.1/24。
- 宿主机 Node 2 上有一个容器 container-2，它的 IP 地址是 100.96.2.3，对应的 docker0 网桥的地址是：100.96.2.1/24。

我们现在的任务，就是让 container-1 访问 container-2。

- 第一步：container-1 容器里的进程发起的 IP 包，其源地址就是 100.96.1.2，目的地址就是 100.96.2.3。此时容器发出的IP包进入宿主机的docker0网桥，从而出现在宿主机。
- 第二步：这时，这个IP包的下一个IP目的地，就取决于宿主机路由规则。此时，Flannel 已经在宿主机上创建出了一系列的路由规则，以 Node 1 为例，如下所示：

```
# 在 Node 1 上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2
```

根据宿主机路由表，目的地址100.96.2.3匹配该表的第二条规则，即flannel0设备。

这个flannel0设备是一个TUN 设备（Tunnel 设备），它的功能是：在操作系统内核和用户应用程序之间传递 IP 包。

- 第三步：当操作系统将一个 IP 包发送给 flannel0 设备之后，flannel0 就会把这个 IP 包，交给创建这个设备的应用程序，也就是 Flannel 进程。这是一个从内核态（Linux 操作系统）向用户态（Flannel 进程）的流动方向。
- 第四步：宿主机上的 flanneld 进程（Flannel 项目在每个宿主机上的主进程），就会收到这个 IP 包。然后，flanneld 看到了这个 IP 包的目的地址，是 100.96.2.3，就把它发送给了 Node 2 宿主机。

这里，flanneld 又是如何知道这个 IP 地址对应的容器，是运行在 Node 2 上的呢？

Flannel 项目里一个非常重要的概念：**子网（Subnet）**。

事实上，在由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”。在我们的例子中，Node 1 的子网是 100.96.1.0/24，container-1 的 IP 地址是 100.96.1.2。Node 2 的子网是 100.96.2.0/24，container-2 的 IP 地址是 100.96.2.3。

而这些子网与宿主机的对应关系，正是保存在 Etcd 当中，如下所示：

```
$ etcdctl ls /coreos.com/network/subnets
/coreos.com/network/subnets/100.96.1.0-24
/coreos.com/network/subnets/100.96.2.0-24
/coreos.com/network/subnets/100.96.3.0-24
```

所以，flanneld 进程在处理由 flannel0 传入的 IP 包时，就可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24），从 Etcd 中找到这个子网对应的宿主机的 IP 地址是 10.168.0.3，如下所示：

```
$ etcdctl get /coreos.com/network/subnets/100.96.2.0-24
{"PublicIP":"10.168.0.3"}
```

对于 flanneld 来说，只要 Node 1 和 Node 2 是互通的，那么 flanneld 作为 Node 1 上的一个普通进程，就一定可以通过上述 IP 地址（10.168.0.3）访问到 Node 2(通过UDP)。

- 第五步：flanneld 在收到 container-1 发给 container-2 的 IP 包之后，就会把这个 IP 包直接封装在一个 UDP 包里，然后发送给 Node 2。

这个 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，则是 container-2 所在的宿主机 Node 2 的地址。

每台宿主机上的 flanneld，都监听着一个 8285 端口，所以 flanneld 只要把 UDP 包发往 Node 2 的 8285 端口即可。通过这样一个普通的、宿主机之间的 UDP 通信，一个 UDP 包就从 Node 1 到达了 Node 2。而 Node 2 上监听 8285 端口的进程也是 flanneld，所以这时候，flanneld 就可以从这个 UDP 包里解析出封装在里面的、container-1 发来的原 IP 包。

- 而接下来 flanneld 的工作就非常简单了：flanneld 会直接把这个 IP 包发送给它所管理的 TUN 设备，即 flannel0 设备。

这正是一个从用户态向内核态的流动方向（Flannel 进程向 TUN 设备发送数据包），所以 Linux 内核网络栈就会负责处理这个 IP 包，具体的处理方法，就是通过本机的路由表来寻找这个 IP 包的下一步流向。

```
# 在 Node 2 上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.2.0
100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.3
```

由于这个 IP 包的目的地址是 100.96.2.3，它跟第三条、也就是 100.96.2.0/24 网段对应的路由规则匹配更加精确。所以，Linux 内核就会按照这条路由规则，把这个 IP 包转发给 docker0 网桥。

到这里，后面就和单宿主机容器通信的后半部分一样了。

需要注意的是，上述流程要正确工作还有一个重要的前提，那就是 docker0 网桥的地址范围必须是 Flannel 为宿主机分配的子网。这个很容易实现，以 Node 1 为例，你只需要给它上面的 Docker Daemon 启动时配置如下所示的 bip 参数即可：

```
$ FLANNEL_SUBNET=100.96.1.1/24
$ dockerd --bip=$FLANNEL_SUBNET ...
```

![](https://tva1.sinaimg.cn/large/008i3skNly1gquqd539kgj31fl0oeac6.jpg)
 
 
可以看到，Flannel UDP 模式提供的其实是一个三层的 Overlay 网络:

它首先对发出端的 IP 包进行 UDP 封装，然后在接收端进行解封装拿到原始的 IP 包，进而把这个 IP 包转发给目标容器。这就好比，Flannel 在不同宿主机上的两个容器之间打通了一条“隧道”，使得这两个容器可以直接使用 IP 地址进行通信，而无需关心容器和宿主机的分布情况。

实际上，相比于两台宿主机之间的直接通信，基于 Flannel UDP 模式的容器通信多了一个额外的步骤，即 flanneld 的处理过程。而这个过程，由于使用到了 flannel0 这个 TUN 设备，仅在发出 IP 包的过程中，就需要经过三次用户态与内核态之间的数据拷贝，如下所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gquqend14bj30oq0gh0t8.jpg)

第一次：用户态的容器进程发出的 IP 包经过 docker0 网桥进入内核态；

第二次：IP 包根据路由表进入 TUN（flannel0）设备，从而回到用户态的 flanneld 进程；

第三次：flanneld 进行 UDP 封包之后重新进入内核态，将 UDP 包通过宿主机的 eth0 发出去。

### 2.1.2 VXLAN

Virtual Extensible LAN（虚拟可扩展局域网)

是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）。

VXLAN 的覆盖网络的设计思想是：

在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。

而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

而 VTEP 设备的作用，其实跟前面的 flanneld 进程非常相似。只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的（因为 VXLAN 本身就是 Linux 内核中的一个模块）。

![](https://tva1.sinaimg.cn/large/008i3skNly1gquqi9orfzj31d30px0t4.jpg)

可以看到，图中每台宿主机上名叫 flannel.1 的设备，就是 VXLAN 所需的 VTEP 设备，它既有 IP 地址，也有 MAC 地址。

现在，我们的 container-1 的 IP 地址是 10.1.15.2，要访问的 container-2 的 IP 地址是 10.1.16.3。

- 第一步：container-1 发出请求之后，这个目的地址是 10.1.16.3 的 IP 包，会先出现在 docker0 网桥，然后被路由到本机 flannel.1 设备进行处理。

也就是说，来到了“隧道”的入口。

为了能够将“原始 IP 包”封装并且发送到正确的宿主机，VXLAN 就需要找到这条“隧道”的出口，**即：目的宿主机的 VTEP 设备。**

而这个设备的信息，正是每台宿主机上的 flanneld 进程负责维护的。

例如：当 Node 2 启动并加入 Flannel 网络之后，**在 Node 1（以及所有其他节点）上，flanneld 就会添加一条如下所示的路由规则：**

```
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
```

这条规则的意思是：凡是发往 10.1.16.0/24 网段的 IP 包，都需要经过 flannel.1 设备发出，并且，它最后被发往的网关地址是：10.1.16.0。(从图 中的 Flannel VXLAN 模式的流程图中我们可以看到，10.1.16.0 正是 Node 2 上的 VTEP 设备（也就是 flannel.1 设备）的 IP 地址。)而这些 VTEP 设备之间，就需要想办法组成一个虚拟的二层网络，即：通过二层数据帧进行通信。

所以在我们的例子中，“源 VTEP 设备”收到“原始 IP 包”后，就要想办法把“原始 IP 包”加上一个目的 MAC 地址(二层数据帧通信需要通过MAC地址)，封装成一个二层数据帧，然后发送给“目的 VTEP 设备”

这里需要解决的问题就是：**“目的 VTEP 设备”的 MAC 地址是什么？**

此时，根据前面的路由记录，我们已经知道了“目的 VTEP 设备”的 IP 地址。而要根据三层 IP 地址查询对应的二层 MAC 地址，这正是 ARP（Address Resolution Protocol ）表的功能。

而这里要用到的 ARP 记录，也是 flanneld 进程在 Node 2 节点启动时，自动添加在 Node 1 上的。我们可以通过 ip 命令看到它，如下所示：

```
# 在 Node 1 上
$ ip neigh show dev flannel.1
10.1.16.0 lladdr 5e:f8:4f:00:e3:37 PERMANENT
```

这条记录的意思非常明确，即：IP 地址 10.1.16.0，对应的 MAC 地址是 5e:f8:4f:00:e3:37。(最新版本的 Flannel 并不依赖 L3 MISS 事件和 ARP 学习，而会在每台节点启动时把它的 VTEP 设备对应的 ARP 记录，直接下放到其他每台宿主机上。)

有了这个“目的 VTEP 设备”的 MAC 地址，Linux 内核就可以开始二层封包工作了。这个二层帧的格式，如下所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gquqnyj58jj30m704zdg1.jpg)

Linux 内核会把“目的 VTEP 设备”的 MAC 地址，填写在图中的 Inner Ethernet Header 字段，得到一个二层数据帧。

但是，上面提到的这些 VTEP 设备的 MAC 地址，对于宿主机网络来说并没有什么实际意义。所以上面封装出来的这个数据帧，并不能在我们的宿主机二层网络里传输。为了方便叙述，我们把它称为“内部数据帧”（Inner Ethernet Frame）。

所以接下来，Linux 内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的 eth0 网卡进行传输。

所以接下来，Linux 内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的 eth0 网卡进行传输。(我们把这次要封装出来的、宿主机对应的数据帧称为“外部数据帧”（Outer Ethernet Frame）。)

为了实现这个“搭便车”的机制，Linux 内核会在“内部数据帧”前面，加上一个特殊的 VXLAN 头，用来表示这个“乘客”实际上是一个 VXLAN 要使用的数据帧。

而这个 VXLAN 头里有一个重要的标志叫作VNI，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。

- 第二步：然后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。

在宿主机看来，它会以为自己的 flannel.1 设备只是在向另外一台宿主机的 flannel.1 设备，发起了一次普通的 UDP 链接。

一个 flannel.1 设备只知道另一端的 flannel.1 设备的 MAC 地址，却不知道对应的宿主机地址是什么。

也就是说，这个 UDP 包该发给哪台宿主机呢？

在这种场景下，flannel.1 设备实际上要扮演一个“网桥”的角色，**在二层网络进行 UDP 包的转发**。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库。

不难想到，这个 flannel.1“网桥”对应的 FDB 信息，也是 flanneld 进程负责维护的。它的内容可以通过 bridge fdb 命令查看到，如下所示：

```
# 在 Node 1 上，使用“目的 VTEP 设备”的 MAC 地址进行查询
$ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```

可以看到，在上面这条 FDB 记录里，指定了这样一条规则，即：

发往我们前面提到的“目的 VTEP 设备”（MAC 地址是 5e:f8:4f:00:e3:37）的二层数据帧，应该通过 flannel.1 设备，发往 IP 地址为 10.168.0.3 的主机。显然，这台主机正是 Node 2，UDP 包要发往的目的地就找到了。

- 第三步：接下来的流程，就是一个正常的、宿主机网络上的封包工作。

我们知道，UDP 包是一个四层数据包，所以 Linux 内核会在它前面加上一个 IP 头，即原理图中的 Outer IP Header，组成一个 IP 包。并且，在这个 IP 头里，会填上前面通过 FDB 查询出来的目的主机的 IP 地址，即 Node 2 的 IP 地址 10.168.0.3。

然后，Linux 内核再在这个 IP 包前面加上二层数据帧头，即原理图中的 Outer Ethernet Header，并把 Node 2 的 MAC 地址填进去。这个 MAC 地址本身，是 Node 1 的 ARP 表要学习的内容，无需 Flannel 维护。这时候，我们封装出来的“外部数据帧”的格式，如下所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gquy2az7bej31fs05cgmd.jpg)

这样，封包工作就宣告完成了。

- 第四步：接下来，Node 1 上的 flannel.1 设备就可以把这个数据帧从 Node 1 的 eth0 网卡发出去。显然，这个帧会经过宿主机网络来到 Node 2 的 eth0 网卡。

这时候，Node 2 的内核网络栈会发现这个数据帧里有 VXLAN Header，并且 VNI=1。所以 Linux 内核会对它进行拆包，拿到里面的内部数据帧，然后根据 VNI 的值，把它交给 Node 2 上的 flannel.1 设备。

而 flannel.1 设备则会进一步拆包，取出“原始 IP 包”。最终，IP 包就进入到了 container-2 容器的 Network Namespace 里。

### 总结

Flannel UDP 和 VXLAN 模式，其实都可以称作“隧道”机制，也是很多其他容器网络插件的基础。比如 Weave 的两种模式，以及 Docker 的 Overlay 模式。

VXLAN 模式组建的覆盖网络，其实就是一个由不同宿主机上的 VTEP 设备，也就是 flannel.1 设备组成的虚拟二层网络。对于 VTEP 设备来说，它发出的“内部数据帧”就仿佛是一直在这个虚拟的二层网络上流动。这，也正是覆盖网络的含义。





