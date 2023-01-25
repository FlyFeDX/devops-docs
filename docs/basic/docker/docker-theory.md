# Docker 原理

<!-- TOC depthFrom:2 depthTo:2 -->

- [一、Docker Namespace](#一docker-Namespace)
- [二、Docker Cgroups](#二docker-Cgroups)
- [三、Docker rootfs](#三docker-rootfs)
- [四、总结](#四总结)

<!-- /TOC -->

容器技术的兴起源于 PaaS 技术的普及；
Docker 公司发布的 Docker 项目具有里程碑式的意义；
Docker 项目通过“容器镜像”，解决了应用打包这个根本性难题。

容器，到底是怎么一回事儿？

容器其实是一种沙盒技术。顾名思义，沙盒就是能够像一个集装箱一样，把你的应用“装”起来的技术。
这样，应用与应用之间，就因为有了边界而不至于相互干扰；而被装进集装箱的应用，也可以被方便地搬来搬去。

## 一、docker Namespace

容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。

对于 Docker 等大多数 Linux 容器来说，Cgroups 技术是用来制造约束的主要手段，而 Namespace 技术则是用来修改进程视图的主要方法。

Linux Namespace（Linux 命名空间）是 Linux 内核（Kernel）提供的功能，它可以隔离一系列的系统资源，如 PID（进程 ID，Process ID）、User ID、Network、文件系统等。

Linux 提供了 3 个系统 API 方便我们使用 Namespace：

- clone () 创建新进程，根据系统调用 flags 来决定哪种类型 Namespace 将会被创建，而该进程的子进程也会包含这些 Namespace。
- setns () 将进程加入到已存在的 Namespace 中。
- unshare () 将进程移出某个 Namespace

Docker 利用 Linux Namespace 功能实现多个 Docker 容器相互隔离，具有独立环境的功能。

## 二、Docker Cgroups

Docker 通过 Linux Namespace 帮进程隔离出自己单独的空间 / 资源，那 Docker 如何限制进程对这些资源的使用呢？

Docker 容器本质依旧是一个进程，多个 Docker 容器运行时，如果其中一个 Docker 进程占用大量 CPU 和内存就会导致其他 Docker 进程响应缓慢，为了避免这种情况，可以通过 Linux Cgroups 技术对资源进行限制。

Linux Cgroups（Linux Contorl Groups，简称 Cgroups）可以对一组进程及这些进程的子进程进行资源限制、控制和统计的能力，其中包括 CPU、内存、存储、网络、设备访问权限等，通过 Cgroups 可以很轻松的限制某个进程的资源占用并且统计该进程的实时使用情况。

Cgroups 由 3 个组件构成，分别是 cgroup（控制组）、subsystem（子系统）以及 hierarchy（层级树），3 者相互协同作用。

- cgroup 是对进程分组管理的一种机制，一个 cgroup 通常包含一组（多个）进程，Cgroups 中的资源控制都以 cgroup 为单位实现。
- subsystem 是一组（多个）资源控制的模块，每个 subsystem 会管理到某个 cgroup 上，对该 cgroup 中的进程做出相应的限制和控制。
- hierarchy 会将一组（多个）cgroup 构建成一个树状结构，Cgropus 可以利用该结构实现继承等功能

Cgroups 会将系统进程分组（cgroup）然后通过 hierachy 构建成独立的树，树的节点就是 cgroup（进程组），每颗树都可以与一个或多个 subsystem 关联，subsystem 会对树中对应的组进行操作。

## 三、Docker rootfs

对 Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：

1. 启用 Linux Namespace 配置；
2. 设置指定的 Cgroups 参数；
3. 切换进程的根目录（Change Root）。

这样，一个完整的容器就诞生了。

Linux 系统中的根文件系统，Root FileSystem，简称为根文件系统 rootfs；根文件系统是包含在与根目录相同的分区上的文件系统，是内核启动时所 mount 的第一个文件系统，内核代码映像文件保存在根文件系统中，而系统引导启动程序会在根文件系统挂载之后, 把初始化脚本和服务等加载到内存中运行。Docker 支持不同的存储驱动，包括 aufs、devicemapper、overlay2、zfs 和 vfs 等，在最新的 Docker 中，overlay2 取代了 aufs 成为了推荐的存储驱动。

rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

所以说，rootfs 只包括了操作系统的“躯壳”，并没有包括操作系统的“灵魂”。

容器的 rootfs 自下而上分为三个部分，只读层(ro+wh)，init 层(ro+wh)，读写层(rw)

- 只读层，对于的 docker 的镜像层, 这部分不可被修改；
- init 层，夹在只读层和读写层之间, Docker 项目单独生成的一个内部层，专门用来存放/etc/hosts、/etc/resolv.conf 等信息；
- 读写层，它的挂载方式为 rw，一旦在容器里做了写操作，你修改产生的内容就会以增量的方式出现在这个层中。比如 foo 文件就会被.wh.foo 文件遮挡。

## 四、总结

![应用管理功能模型](../../../images/modules/docker全景图.png)

这个容器进程“python app.py”，运行在由 Linux Namespace 和 Cgroups 构成的隔离环境里；而它运行所需要的各种文件，比如 python，app.py，以及整个操作系统文件，则由多个联合挂载在一起的 rootfs 层提供。

这些 rootfs 层的最下层，是来自 Docker 镜像的只读层。

在只读层之上，是 Docker 自己添加的 Init 层，用来存放被临时修改过的 /etc/hosts 等文件。

而 rootfs 的最上层是一个可读写层，它以 Copy-on-Write 的方式存放任何对只读层的修改，容器声明的 Volume 的挂载点，也出现在这一层。
