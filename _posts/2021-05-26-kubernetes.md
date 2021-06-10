---
layout:     post
title:      kubernetes基本概念
subtitle:   kubernetes基本概念
date:       2021-05-26
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - kubernetes
---

# 架构资源

Kubernetes Cluster由Master和Node组成，节点上运行着若干Kubernetes服务。

## 1.1 Master节点 

Master是Kubernetes Cluster的大脑，运行着的Daemon服务包括kube-apiserver、kube-scheduler、kube-controller-manager、etcd和Pod网络（例如flannel）

![](https://tva1.sinaimg.cn/large/008i3skNly1gqvu36maiij30g10koaad.jpg)

- API Server(kube-apiServer)

APIServer提供HTTP/HTTPS Restful API,各种客户端工具(CLI或UI)以及Kubernetes 其他组件可以通过它管理Cluster的各种资源。

- Scheduler(Kube-scheduler)

Scheduler负责决定将Pod放在哪个Node上运行。

Scheduler在调度时会充分考虑Cluster的拓扑结构，当前各个节点的负载，以及应用对高可用，性能，数据亲和性的需求。

- Controller Manager(Kube-controller-manager)

Controller Manager负责管理Cluster各种资源，保证资源处于预期状态。

Controller Manager 由多种controller组成，包括：replication controller，endponits controller，namespace controller，serviceaccount controller等。

不同的controller管理不同的资源。例如，replication controller管理Deployment，StatefulSet，DaemonSet的声明周期。namespace controller 管理Namespace资源。

- etcd

etcd负责保存Kubernets Cluster的配置信息和各种资源的状态信息。当数据发生变化时，etcd会快速地通知Kubernetes相关组件。

- Pod网络

Pod要能够相互通信，Kubernetes Cluster必须部署Pod网络，flannel是其中一个可选方案。

## 1.2 Node节点

Node是Pod运行的地方，Kubernetes支持Docker，rkt等容器runtime。Node上运行的Kubernetes组件有Kubelet，kube-proxy和Pod网络(例如flannel)。

![](https://tva1.sinaimg.cn/large/008i3skNly1gqw1dnnverj30g50l70ss.jpg)

- kubelet

kubelet是Node的agent，当Scheduler确定在某个Node上运行Pod后，会将Pod的具体配置信息(image,volume等)发送给该节点的kubelet，kubelet根据这些信息创建和运行容器，并想Master报告运行状态。

- kube-proxy

service在逻辑上代表了后端的多个Pod，外界通过service访问Pod。kube-proxy负责将service的请求转发到Pod。

每个Node都会运行kube-proxy服务，它负责访问service的TCP/UDP数据流转发到后端的容器。如果有多个副本，kube-proxy会实现负载均衡。

- Pod网络

执行以下命令查看集群中的node：kubectl get nodes

```
[root@tj1-b2c-systech-mione-test02 ~]# kubectl get nodes
NAME                                STATUS   ROLES                  AGE   VERSION
tj1-b2c-systech-mione-test01.kscn   Ready    worker                 3d    v1.21.1
tj1-b2c-systech-mione-test02.kscn   Ready    control-plane,master   8d    v1.21.1
```

查看node的详细信息：kubectl describe ndoe <node_name>

```
[root@tj1-b2c-systech-mione-test02 ~]# kubectl describe node tj1-b2c-systech-mione-test02
Name:               tj1-b2c-systech-mione-test02.kscn // Node名称
Roles:              control-plane,master  
Labels:             beta.kubernetes.io/arch=amd64 // 标签
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=tj1-b2c-systech-mione-test02.kscn
                    kubernetes.io/os=linux
                    name=tom
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
                    node.kubernetes.io/exclude-from-external-load-balancers=
                    role=master
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"b6:44:b4:49:e3:89"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.38.167.198
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 19 May 2021 11:22:49 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  tj1-b2c-systech-mione-test02.kscn
  AcquireTime:     <unset>
  RenewTime:       Thu, 10 Jun 2021 20:54:31 +0800 
Conditions: // Node当前运行状态(磁盘，内存，网络，pid等，一切正常则Ready为true)
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 10 Jun 2021 16:32:31 +0800   Thu, 10 Jun 2021 16:32:31 +0800   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Thu, 10 Jun 2021 20:53:50 +0800   Wed, 19 May 2021 11:22:44 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 10 Jun 2021 20:53:50 +0800   Wed, 19 May 2021 11:22:44 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 10 Jun 2021 20:53:50 +0800   Wed, 19 May 2021 11:22:44 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 10 Jun 2021 20:53:50 +0800   Wed, 19 May 2021 11:23:59 +0800   KubeletReady                 kubelet is posting ready status
Addresses: // Node的主机名与地址
  InternalIP:  10.38.167.198  
  Hostname:    tj1-b2c-systech-mione-test02.kscn
Capacity: // Node的资源数量，包括CPU，内存，最大可调度Pod数量等
  cpu:                40
  ephemeral-storage:  298309296Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131647820Ki
  pods:               110
Allocatable:
  cpu:                40
  ephemeral-storage:  274921846739
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131545420Ki
  pods:               110
System Info: // 主机的信息：主机ID，系统UUID， Linux内核版本，操作系统类型及版本，kubelet与kube-proxy版本号
  Machine ID:                 12bd464795144819bdfb5e8b09e72e63
  System UUID:                20FE1E2F-743A-04D5-E611-B7CF00CD874F
  Boot ID:                    ed9340d7-f4ac-4024-b512-3c5d5c01fb3b
  Kernel Version:             4.9.2-3.0.0.std7b.el7.5.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.8
  Kubelet Version:            v1.21.1
  Kube-Proxy Version:         v1.21.1
PodCIDR:                      10.244.0.0/24 
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (20 in total) //当前Pod列表概要信息
  Namespace                   Name                                                         CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                         ------------  ----------  ---------------  -------------  ---
  default                     hello-minikube-6ddfcc9757-qbtlt                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         22d
  default                     k8s-demo-deployment-65cbbfb876-kzm7g                         10m (0%)      1 (2%)      64Mi (0%)        128Mi (0%)     13d
  default                     k8s-demo-deployment-65cbbfb876-qpb7q                         10m (0%)      1 (2%)      64Mi (0%)        128Mi (0%)     13d
  default                     k8s-demo-deployment-65cbbfb876-zwpls                         10m (0%)      1 (2%)      64Mi (0%)        128Mi (0%)     13d
  default                     mysql-deployment-5cb6f7f865-sgqrx                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         13d
  default                     prometheus-76c67b548f-ss5js                                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         15m
  default                     redis-deployment-8db785d9d-zhqtz                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         17d
  default                     redis-master-rxxdb                                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         17d
  kube-system                 coredns-558bd4d5db-5kk2r                                     100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     22d
  kube-system                 coredns-558bd4d5db-xxxzz                                     100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     22d
  kube-system                 etcd-tj1-b2c-systech-mione-test02.kscn                       100m (0%)     0 (0%)      100Mi (0%)       0 (0%)         22d
  kube-system                 kube-apiserver-tj1-b2c-systech-mione-test02.kscn             250m (0%)     0 (0%)      0 (0%)           0 (0%)         22d
  kube-system                 kube-controller-manager-tj1-b2c-systech-mione-test02.kscn    200m (0%)     0 (0%)      0 (0%)           0 (0%)         22d
  kube-system                 kube-flannel-ds-7gvwm                                        100m (0%)     100m (0%)   50Mi (0%)        50Mi (0%)      22d
  kube-system                 kube-proxy-ncdsl                                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         22d
  kube-system                 kube-scheduler-tj1-b2c-systech-mione-test02.kscn             100m (0%)     0 (0%)      0 (0%)           0 (0%)         22d
  kubernetes-dashboard        dashboard-metrics-scraper-5594697f48-qzf4d                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         22d
  kubernetes-dashboard        kubernetes-dashboard-57c9bfc8c8-628pt                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         22d
  metallb-system              controller-fb659dc8-thgfk                                    100m (0%)     100m (0%)   100Mi (0%)       100Mi (0%)     14d
  metallb-system              speaker-fzg6w                                                100m (0%)     100m (0%)   100Mi (0%)       100Mi (0%)     14d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                1180m (2%)  3300m (8%)
  memory             682Mi (0%)  974Mi (0%)
  ephemeral-storage  100Mi (0%)  0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:              <none>
```

- Node主机地址与主机名
- Node上资源数量，CPU，内存，最大可调度Pod
- 主机系统信息：主机ID，系统UUID，操作系统版本，Docker版本，kubelet与kube-proxy版本号
- 当前运行Pod列表
- 已分配资源使用概要
- Node相关Event信息

## 1.3 Pod

![](https://tva1.sinaimg.cn/large/008i3skNly1gqxzh5mnmfj30s50lcwf8.jpg)

每个Pod都有一个特殊的"根容器"Pause容器以及多个用户容器。该不易死亡的Pause容器代表整个容器组的状态。

Kubernetes为每个Pod分配了一个唯一的IP地址，一个Pod里的多个容器共享PodIP，共享Pause容器挂载的Volume。

Pod具有两种类型：普通Pod和静态Pod。

- 静态Pod被放在某个具体Node的具体文件里，并且只在此Node上启动，运行(没有放在etcd)。
- 而普通Pod一旦被创建，就被放入etcd中存储。随后被k8s Master调度到具体Node并与之绑定，随后该Pod被Node上的kubelet进程实例化成一组Docker容器并启动。(默认情况下，当Pod里的某个容器停止时，k8s会自动检测到该问题，并重启这个Pod，即重启Pod中的所有容器。如果Pod所在的Node宕机，就会将这个Node上的所有Pod重新调度到其他节点。)


以下例子定义了一个mysql pod
```
apiVersion:v1
kind:Pod
metadata:
    name:myweb
    labels:
        name:myweb
spec:
    containers:
    - name:myweb
      image:mysql
      ports:
      - containerPort:8080
      env:
      - name:MYSQL_SERVICE_HOST
      value:'mysql'
      - name:MYSQL_SERVICE_PORT
      value:'3306'
```

- kind:pod :这是一个Pod定义
- metadata.name:Pod的名称
- metadata.labels:资源对象的标签，这里声明myweby拥有一个name=myweb的标签
- Pod里所包含的的容器组的定义在spec中声明，这里定义了一个名为myweb，对应镜像为mysql的容器，该容器注入了名为MYSQL_SERVICE_HOST='mysql'和MYSQL_SERVICE_PORT='3306'的环境变量。并且在8080端口启动进程。

Pod的IP加上这里的容器端口(containerPort),组成了一个新的概念：Endpoint。它代表Pod里一个服务进程对外通信的地址。

Pod也具有相应的一些Event概念，可以通过kubectl describe pod xxx查看

```
[root@tj1-b2c-systech-mione-test02 ~]# kubectl describe pod cpu-ram-demo
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  22m   default-scheduler  Successfully assigned default/cpu-ram-demo to tj1-b2c-systech-mione-test01.kscn
  Normal  Pulling    22m   kubelet            Pulling image "wodiwudi/kubia"
  Normal  Pulled     21m   kubelet            Successfully pulled image "wodiwudi/kubia" in 5.010250193s
  Normal  Created    21m   kubelet            Created container cpu-ram-demo-container
  Normal  Started    21m   kubelet            Started container cpu-ram-demo-container
```

每个Pod都可以对其能使用的CPU上的计算资源设置限额，当前可以设置限额的计算资源有CPU和Memory两种。

其中CPU的资源单位为CPU(Core)的数量，是一个绝对值。k8s中以千分之一进行配额(CPU为100-300m,即表示占用0.1-0.3个U，而不论宿主机有多个CPU)。

- Requests：该资源的最少申请量，系统必须满足要求
- Limits：该资源最大允许使用量，不能被突破，当容器试图使用超过这个量的资源时，会被Kubernetes杀掉并重启。

通常我们会把Request设置成一个较小的数值，符合容器平时的工作负载情况下的资源需求，而把Limit设置为峰值负载情况下资源占用最大量。

```
spec:
    containers:
    - name:db
      image:mysql
      resources:
        requests:
          memory:"64Mi" // 64mb内存
          cpu:"250m" // 0.25个CPU
        limits:
          memory:"128Mi" // 128mb内存
          cpu:"500m" // 0.5个cpu
```

![](https://tva1.sinaimg.cn/large/008i3skNly1gqy3ho7yadj30w00gfaan.jpg)

## 1.4 Label

一个Label是一个key-value的键值对，其中key和value都是用户进行指定。Label可以被附加到任何资源对象上，如Node，Pod，Service，RC等。

一个资源对象可以定义任意数量的Label。同一个Lable也可以添加到任意数量的资源对象上。

Label通常可以在资源对象定义时确定，也可以在资源对象创建后动态添加或删除。

Label Selector：

- name=redis-slave:匹配所有具有标签name=redis-slave的资源对象
- env!=production:匹配所有不具有标签env=production的资源对象
- name in (redis-master,redis-slave)
- name not in (php-frontend)
- name=redis-slave,env!=production // 组合筛选，用，分割

Label Selector的使用场景：

- kube-controller进程通过在资源对象RC上定义的Label Selector来筛选要监控的Pod副本数量
- kube-proxy进程通过Service的Label Selectorl来选择对应的Pod，自动建立每个Service到对应Pod的请求转发路由表
- 通过对某些Node定义特定的Label，并且在Pod定义文件中使用NodeSelector标签调度策略，kube-scheduler进程可以实现Pod定向调度。


eg:

```
apiVersion: v1
kind:Pod
metadata:
    name: myweb
    labels:
        app: myweb
```

```
apiVersion: v1
kind: ReplicationController
metadata:
    name: myweb
spec:
    replicas: 1
    selector:
        app: myweb
    template:
    ...
```

或者：

```
apiVersion: v1
kind: ReplicationController
metadata:
    name: myweb
spec:
    replicas: 1
    selector:
        matchLabels: // matchLabels用来定义一组label，与直接写在selector中相同
            app: myweb
    template:
    ...
```

Label selctor的使用场景：

- kube-controller进程通过在资源对象RC上定义Label Selector来筛选要监控的副本数量
- kube-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立每个Service到对应的Pod的请求转发路由表，从而实现Service的负载均衡
- 通过对某些Node定义Label，并且在Pod定义中使用NodeSelector这种标签调度策略，kube-scheduler进程可以实现Pod定向调度。


## 1.5 Replication Controller

它定义了一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合期望值

其定义主要包含以下部分：

- Pod期望的副本数量
- 用于筛选目标Pod的Label Selector
- 当Pod的副本数量小于期望值时，用于创建新Pod的Pod模版。

```
apiVersion:v1
Kind:ReplicationController
metadata:
    name:frontend // rc名
spec:
    replicas:1 // 副本数
    selector:
        tier:frontend //需要选择的Pod应该具有的标签
    template: // 当pod数不足时，依照以下模版创建
        metadata: // 模版的元数据，即为pod定义一些标签，需要与上面的selector对应上
          labels:app-demo 
          tier:frontend
        spec: 
          containers: // Pod内定义容器相关
          - name:tomcat-demo // 容器名
          image:tomcat // 镜像选择
          imagePullPolicy:IfNotPresent // 拉取镜像策略
          env: // 为容器设置环境变量
          - name:GET-HOSTS-FROM // 环境变量名
          value:dns // 环境变量名
          ports: // Pod相关端口定义
          - containerPort:80 // 定义Pod内容器端口
```

我们定义了一个RC并提交到Kubernetes集群后，Master上的Controller Manager组件就会得到通知，定期巡检系统中当前存活的目标Pod，并确保目标Pod实例数量刚好等于此RCd的期望值。

在运行时，我们也可以动态的修改RC的副本数量，来实现Pod的动态缩放。(使用kubectl scale命令)

```
kubectl scale rc rc名 --replicas=3
```

为了删除所有Pod，可以设置replicas=0，然后更新RC。另外也可以使用delete命令

```
kubectl scale rc redis-slave --replicas=3 // 其中，redis-slave为Pod名
```

需要注意的是，删除RC并不会影响通过RC已经创建好的Pod。为了删除所有Pod，可以设置replicas的值为0，然后更新该RC。另外，kubectl提供了stop和delete命令来一次性RC和RC控制的全部Pod。

## 1.6 Replica Set

Replica Set支持基于集合的Label selector。

## 1.7 Deployment

Deployment可以任务是RC的一个升级，且它可以随时知道当前Pod部署的进度。

```
apiVersion: v1
kind: Deployment
metadata:
    name:fronted
spec:
    replicas: 1
    selector:
        tier:fronted
template:
    metadata:
    labels:
        app:app-demo
        tier:fronted
    spec:
        containers:
        - name: tomcat-demo
          image: tomcat
          imagePullPolicy:IfNotPresent
          ports:
          - containerPort:8080
```

```
kubectl apply -f tomcat-deployment.yaml
uarealoser@uarealoserdeMacBook-Pro k8s % kubectl get deployment 
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE 
gwdash-fe-test               1/1     1            1           6h42m
gwdash-test                  1/1     1            1           25h
mionedemo-30560-deployment   2/2     2            2           30h
```

## 1.8 StatefulSet

主要面向有状态的Pod，具有如下特性：

- StatefulSet里每个Pod都有稳定唯一的网络标识，可以用来发现集群中的其他成员。
- StatefulSet里Pod副本的启停顺序是受控制的。
- StatefulSet里的Pod采用稳定的持久化存储卷，通过PV或PVC实现。

StatefulSet与PV卷捆绑使用，以存储Pod的状态数据，与Headless Service配合使用，即需要在每个StatefulSet中声明属于那个Headless Service(Headless Service没有cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表)StatefulSet在Headless基础上又为StatefulSet控制的每个Pod实例都创建了DNS域名，其格式为:"$(podname).$(headless service name)"。

例如：3个节点的kafka的StatefulSet集群，对应的headless service名称为kafka，StatefulSet的名称为kafka，则StatefulSetl里的三个Pod的DNS分别为kafka-0.kafka,kafka-1.kafka,kafka-2.kafka。这些DNS名称可以直接在配置文件中固定下来。

## 1.9 Service

Service服务是Kunernetesl里最核心的资源对象之一，每个Service就可以抽象为一个微服务。

![](https://tva1.sinaimg.cn/large/008i3skNly1gqy6t00eayj30qk0azt9x.jpg)

从图中可以看到，Kubernetes的service定义了一个服务的访问入口地址，前端的应用(Pod)通过这个入口地址访问其背后的一组Pod副本组成的集群实例，Service与其后端的Pod副本集群之间则是通过Label Selector来实现无缝对接的。RC的作用实际是保证Service的服务能力和服务质量符合预期标准。

即：为这组Pod开启一个对外端口如8000，并将这些Pod的endpoint(ip:port)列表加入这个8000端口的转发列表，客户端可以这个service的对外ip:8000来访问后端这些服务，并做自动负载均衡(kube-proxy实现)。

这里，Service并没有共享这个负载均衡器(kube-proxy)地址，而是为每个Service都分配一个全局唯一的虚拟IP地址，这个虚拟IP地址被称为Cluster IP。在Service的整个生命周期内，它的Cluster Ip并不会改变。(因此服务发现就可以用service nameh和cluster ip做一个dns映射)

```
apiVersion: v1
kind: Service
metadata:
    name: tomcat-service
spec:
    ports:
    - port: 8080
    selector:
        tier:fronted
```

在spec.ports的定义中，targetPort属性用来确定提供该服务的容器所暴露的端口号，port属性则定义了Service的虚拟端口号，当没有指定targetPort时，则表示targetPort与prot相同。

使用kubectl get endpoints即可看见pod集群的Ip:port

```
[root@tj1-b2c-systech-mione-test02 ~]# kubectl get endpoints
NAME                 ENDPOINTS                                                        AGE
example-service      10.244.1.64:80,10.244.1.65:80                                    14d
hello-minikube       10.244.0.6:8080                                                  15d
kubernetes           10.38.167.198:6443                                               22d
```
查看clusterIP：

```
kubectl get svc example-service -o yaml
```

clusterIp无法被ping，因为没有一个"实体网络对象"来响应。

### 1.9.1 多端口service

k8s service支持多个endpoint，在存在多个endpoint的情况下，要求每个endpoint都定义一个名称来区分。

```
apiVersion: v1
kind: Service
metadata:
    name: tomcat-service
spec:
    ports:
    - port:8080
    name:service-port
    - port:8081
    name:admin-port
    selector:
        tier:fronted
```

k8s的服务发现机制：

之所以需要为每个端口定义一个名称，主要用于k8s的服务发现机制。早起采用环境变量的形式在创建Pod的时候写入Pod中，后采用DNS方式：把服务名作为DNS域名，这样程序就可以用服务名来建立通信了。？

### 1.9.2 端口外访问Service

- 采用NodePort

```
apiVersion: v1
kind: Service
metadata:
    name: tomcat-service
spec:
    type: NodePort
    ports:
    - port:8080
      nodePort:31002
    selector:
        tier:fronted
```

这里nodePort:31002表明手动为Service端口映射指定一个ip，否则会自动分配一个ip。

- loadbalancer

但是这种方式并没有完全解决外部访问Service的问题，例如负载均衡问题。(例如我们的集群有10个Node，则此时最好有一个负载均衡器，外部的请求只需要访问此负载均衡器的IP地址，由负载均衡器来负责转发流量到后面的某个Node的NodePort)

但是，如果是手动运行负载均衡实例(例如配置nginx)来进行负载均衡，会增大出错的概率。

如果集群运行在公有云上，则将Service的type改为"type=LoadBalancer",k8s就会自动创建一个对应的load balancer实例，并返回它的IP地址供外部客户端调用。