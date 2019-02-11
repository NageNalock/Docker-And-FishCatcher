# Docker-And-FishCatcher

容器实现的两种最关键技术: Namespace, Cgroups

Dokcer 总结:

Docker 技术: 一种使用了 Namespace, Cgroups, 以及 Change root 来实现的沙盘技术

Docker 镜像: 本质上一个 rootfs( 根文件系统)

- NameSpace 隔离其他进程信息
- Cgroups 限制进程资源
- ChangeRoot 实现自己的根文件系统(采用的联合文件系统, 即增量的根文件系统, 防止每次使用镜像都需要重新制作一个根文件系统)(为什么不在 amount namespace 来限制, 因为 amount 只会在挂载时生效一次, 我们再打开时就不会挂载了)

## 容器

从过去以物理机和虚拟机为主体的开发运维环境，向以容器为核心的基础设施的转变过程，并不是一次温和的改革，而是涵盖了对网络、存储、调度、操作系统、分布式原理等各个方面的容器化理解和改造。

**容器**其实是一种沙盒技术。顾名思义，沙盒就是能够像一个集装箱一样，把你的应用“装”起来的技术。这样，应用与应用之间，就因为有了边界而不至于相互干扰；而被装进集装箱的应用，也可以被方便地搬来搬去，这不就是 PaaS 最理想的状态嘛。

## 编排(Fig 版)

其实，“编排”（Orchestration）在云计算行业里不算是新词汇，它主要是指用户如何通过某些工具或者配置来完成一组虚拟机以及关联资源的定义、配置、创建、删除等工作，然后由云计算平台按照这些指定的逻辑来完成的过程。

而容器时代，“编排”显然就是对 Docker 容器的一系列定义、配置和创建动作的管理。而 Fig 的工作实际上非常简单：假如现在用户需要部署的是应用容器 A、数据库容器 B、负载均衡容器 C，那么 Fig 就允许用户把 A、B、C 三个容器定义在一个配置文件中，并且可以指定它们之间的关联关系，比如容器 A 需要访问数据库容器 B。

------

- 容器技术的兴起源于 PaaS 技术的普及；
- Docker 公司发布的 Docker 项目具有里程碑式的意义；
- Docker 项目通过“容器镜像”，解决了应用打包这个根本性难题。

**容器本身没有价值，有价值的是“容器编排”。**

对于进程来说，它的**静态表现**就是**程序**，平常都安安静静地待在磁盘上；而一旦运行起来，它就变成了计算机里的**数据和状态的总和**，这就是它的动态表现。

## Namespace

每当我们在宿主机上运行了一个 /bin/sh 程序，操作系统都会给它分配一个进程编号，比如 PID=100。这个编号是进程的唯一标识，就像员工的工牌一样。所以 PID=100，可以粗略地理解为这个 /bin/sh 是我们公司里的第 100 号员工，而第 1 号员工就自然是比尔 · 盖茨这样统领全局的人物

而现在，我们要通过 Docker 把这个 /bin/sh 程序运行在一个容器当中。这时候，Docker 就会在这个第 100 号员工入职时给他施一个“障眼法”，让他永远看不到前面的其他 99 个员工，更看不到比尔 · 盖茨。这样，他就会错误地以为自己就是公司里的第 1 号员工。

这种机制，其实就是对**被隔离应用**的**进程空间**做了手脚，使得这些进程只能**看到**重新计算过的**进程编号**，比如 PID=1。可实际上，**他们在宿主机的操作系统里，还是原来的第 100 号进程**。

**这种技术，就是 Linux 里面的 Namespace 机制**(只是针对 pid 的 Namespace, 还有挂载点, 用户等Namespace), 一个**障眼法**

所以，Docker 容器这个听起来玄而又玄的概念，实际上是**在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数**。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了

**所以说，容器，其实是一种特殊的进程而已。**

> **自己的总结**
>
> 容器技术是一种**沙盘**技术
>
> 类似于**一种虚拟机**, 其实表现为一种配置了一组 **Namespace** 参数的特殊的**进程**

## 容器与虚拟机的对比

**共同点**: 都是将**进程划分一个独立空间**

以下是一个虚拟机与容器对比图(**其实不准确**)

![img](https://static001.geekbang.org/resource/image/80/40/8089934bedd326703bf5fa6cf70f9740.png)



- 左边是**虚拟机**:
  - 名为 **Hypervisor** 的软件是虚拟机**最主要的部分**: 
    - 它通过硬件虚拟化功能，模拟出了运行一个操作系统需要的各种硬件，比如 CPU、内存、I/O 设备等等。
    - 然后，它在这些虚拟的硬件上安装了一个新的操作系统，即 Guest OS。
  - 用户的应用进程就可以运行在这个虚拟的机器中，它能**看到的自然也只有 Guest OS 的文件和目录**，以及这个机器里的虚拟设备。这就是为什么虚拟机也能起到将不同的应用进程相互隔离的作用。
- 右边是 **Docker**:
  - 一个名为 **Docker Engine** 的软件替换了 Hypervisor。这也是为什么，很多人会把 Docker 项目称为“轻量级”虚拟化技术的原因，实际上就是把虚拟机的概念套在了容器上
  - **但是这是十分不准确的**

> 跟真实存在的虚拟机不同，在使用 Docker 的时候，**并没有一个真正的“Docker 容器”运行在宿主机里面**。Docker 项目帮助用户启动的，**还是原来的应用进程**，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数。

在之前虚拟机与容器技术的对比图里，**不应该把 Docker Engine 或者任何容器管理工具放在跟 Hypervisor 相同的位置**，因为它们并不像 Hypervisor 那样对应用进程的**隔离**环境负责，也不会创建任何实体的“容器”，真正对隔离环境负责的是宿主机操作系统本身

因此, 对比图实际上应该是如下图

![img](https://static001.geekbang.org/resource/image/9f/59/9f973d5d0faab7c6361b2b67800d0e59.jpg)

我们应该把 Docker 画在跟应用同级别并且靠边的位置。这意味着，**用户运行在容器里的应用进程，跟宿主机上的其他进程一样，都由宿主机操作系统(还是由操作系统管理)统一管理**，只不过这些被隔离的进程拥有额外设置过的 Namespace 参数。而 Docker 项目在这里扮演的角色，更多的是旁路式的**辅助和管理**工作。

而相比之下，**容器化后**的用户应用，却依然还是一个宿主机上的普通进程，这就意味着这些(**优点**)

- 因为虚拟化而带来的性能损耗都是不存在的；
- 而另一方面，使用 Namespace 作为隔离手段的容器并不需要单独的 Guest OS，这就使得容器额外的资源占用几乎可以忽略不计
- 用俩词概括的话就是, **敏捷**与**高性能**

**缺点**: **隔离的不彻底**

- 首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作**系统内核**。

  举个例子, 你不能在 Windows 主机上运行 Linux 容器

- 其次，在 Linux 内核中，有很多资源和对象是**不能被 Namespace 化**的，最典型的例子就是：时间。

**后果**:

容器的被攻击面非常大, 一般来说**没有谁敢直接将 Linux 容器直接暴露在公网中**(secomp , 一种流量管理功能 ,可以起到一定的防范作用, )

## 对容器的限制(Linux Cgroups)

举个例子:

虽然容器内的第 1 号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上，它作为第 100 号进程与其他所有进程之间依然是平等的竞争关系。这就意味着，虽然第 100 号进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是**可以随时被宿主机上的其他进程（或者其他容器）占用的**。当然，这个 100 号进程自己也可能把所有资源吃光。这些情况，显然都不是一个“沙盒”应该表现出来的合理行为。

这就用了, **Linux Cgroups** 就是 Linux 内核中用来**为进程设置资源限制**的一个重要功能。

Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是**限制一个进程组能够使用的资源上限**，包括 CPU、内存、磁盘、网络带宽等等。

简单粗暴地理解呢，**它就是一个子系统目录加上一组资源限制文件的组合**

使用过程:

- 在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫**子系统**。这些都是我这台机器当前**可以被 Cgroups 进行限制的资源种类**。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法。

- 举个例子, 在 CPU 子系统下, 有 cfs_period 和 cfs_quota 这样的关键词。这两个参数需要组合使用，可以**用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间**。

- 在对应的子系统下面创建一个目录，比如，我们现在进入 /sys/fs/cgroup/cpu 目录(CPU **子系统**)下,创建子目录

- 这个目录就称为一个“控制组”。你会发现，操作系统会在你新创建的 container 目录下，**自动生成**该子系统对应的资源限制文件。

- 我们可以通过查看 container 目录下的文件，看到 container 控制组里的 CPU quota 还没有任何限制（即：-1），CPU period 则是默认的 **100  ms**（100000  us）

- 比如，向 container 组里的 cfs_quota 文件写入 20  ms（20000  us）：它意味着在每 **100  ms 的时间里**，被该控制组限制的进程只能使用 **20  ms 的 CPU 时间**，

  即这个进程只能使用到 20% 的 CPU 带宽。

- 接下来，我们把被限制的进程的 PID 写入 container 组里的 **tasks** 文件，上面的设置就会对该进程生效了

此外, 还可以限制**I/O, CPU 节点(核数), 内存用量**等参数

而**对于 Docker** 等 Linux 容器项目来说,

它们只需要在每个子系统下面，**为每个容器创建一个控制组**（即创建一个新目录），然后在**启动容器进程之后**，把**这个进程的 PID 填写到对应控制组的 tasks 文件**中就可以了。

## 容器的一致性

容器的完整步骤,其实是为待创建的进程

- 启用 Linux Namespace 配置；
- 设置指定的 Cgroups 参数；
- 切换进程的根目录（Change Root）(多了这一步)

因为 Mount Namespace 只在**挂载**时生效, 为了使容器**只能看见其自己的文件目录结构**, 需要 **chroot** 命令(最后一步)

而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的**文件系统**，就是所谓的**“容器镜像”**。它还有一个更为专业的名字，叫作：**rootfs（根文件系统）**。也是解决**容器一致性问题**的方案

#### 一致性

本来对于 PaaS 的部署来说, **打包**是一个痛苦的过程

**由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。**

#### 打包时的一个问题

- 不过，这时你可能已经发现了另一个非常棘手的问题：难道我每开发一个应用，或者升级一下现有的应用，都要重复制作一次 rootfs 吗？

  答: 可以对**每一步有意义的**操作来制作一个 rootfs, 但是这样会带来 rootfs 的碎片化.

- **那么**，既然这些修改都基于一个旧的 rootfs，我们能不能以增量的方式去做这些修改呢？这样做的好处是，所有人都**只需要维护相对于 base rootfs 修改的增量内容**，而不是每次修改都制造一个“fork”。

  答:Docker 在镜像的设计中，引入了**层（layer）**的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个**增量 rootfs**。(使用了**联合文件系统**)

#### 联合文件系统

Union File System 也叫 UnionFS，最主要的功能是**将多个不同位置的目录联合挂载（union mount）到同一个目录下**。比如，我现在有两个目录 A 和 B，它们分别有两个文件：

```shell
$ tree
.
├── A
│  ├── a
│  └── x
└── B
  ├── b
  └── x
```

然后，我使用联合挂载的方式，将这两个目录挂载到一个公共的目录 C 上：

```shell
$ mkdir C
$ mount -t aufs -o dirs=./A:./B none ./C
```

这时，我再查看目录 C 的内容，就能看到目录 A 和 B 下的文件被合并到了一起(新增了一个文件 C)：

```
$ tree ./C
./C
├── a
├── b
└── x
```

可以看到，在这个合并后的目录 C 里，有 a、b、x 三个文件，**并且 x 文件只有一份**。这，就是“**合并**”的含义。此外，**如果你在目录 C 里对 a、b、x 文件做修改，这些修改也会在对应的目录 A、B 中生效**。

**举个例子**

启动容器: Docker 会从 Docker Hub 上拉取一个 Ubuntu 镜像到本地

这个镜像本质上还是一个 rootfs, 但是是由**多个层**组成

## Docker 

- 根文件系统: 容器的静态视图
- NameSpace + Cgroups 容器的动态视图

作为一个开发者, 我们关心的是容器镜像, 而不是容器的运行状态