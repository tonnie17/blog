+++
title = "20分钟极简入门Docker"
date = 2019-10-12T00:00:00+08:00
tags = ["docker"]
categories = [""]
draft = false
+++

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-75743295e99c473d06c1cfcf98d18afd_1440w.jpg)

## 什么是Docker

首先来介绍一下什么是Docker，Docker是早于2013年发布的开源项目，它借助操作系统的虚拟化技术来实现应用间的资源隔离，从而应用能更加快速方便地打包和部署在任何地方。根据官网描述，Docker是一个借助容器进行开发，部署和运行应用的工具，通俗来说，Docker容器好比一个集装箱一样，里面存放了应用所需要的文件和依赖，这种把应用标准化的过程被叫做为“容器化”。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4864204d38bc146ad21a606ad9a7a7b3_1440w.jpg)

## Docker适合做什么

对于开发人员来说，容器技术为应用的部署提供了沙盒环境，开发者可以在独立的容器运行和管理应用程序进程，Docker提供的抽象层使得开发人员之间可以保持开发环境相对的一致，因此Docker适合用于应用隔离，搭建沙箱环境，此外，由于Docker容器的轻量化，它也被适用于进行持续集成和持续部署。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-f394be6e85acacc8985d63f5c97af86d_1440w.jpg)

## Container VS VM

人们经常用虚拟机和容器来做比较，那么它们两之间有什么区别呢？其实它们最核心的区别在于虚拟化资源的层面不一样，虚拟机是在硬件层之上实现虚拟化的，而容器则直接在操作系统之上实现虚拟化，从图中可以看出，每个虚拟机都需要在一个Guest OS之上，而容器则可以都处在同一个宿主机之上。

因为容器没有虚拟机造成的额外损耗，所以与虚拟机对比，容器不仅运行效率更高，而且部署速度也更快。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-7fc22a7c69ab1c5b3ac3e3414c00ed28_1440w.jpg)

## Docker Engine

Docker Engine是用于运行和编排容器的基础设施工具，我们平时说到的Docker大多数指的是Docker Engine，也就是在命令行和Docker进行交互的时候打交道的后台进程。

这是Docker Engine目前的架构，Docker客户端通过REST API与Docker Daemon来进行交互，Daemon把命令下发给containerd，containerd负责容器的生命周期管理以及镜像管理等，而runc负责创建容器。

Docker首次发布时，Docker Engine由两个核心组件构成：LXC和Docker Daemon。Docker Daemon是单一的二进制文件，包含诸如 Docker客户端、Docker API、容器运行时、镜像构建等。LXC提供了对Namespace(资源隔离)和CGroup(资源限制)等基础工具的操作能力，它们是基于Linux内核的容器虚拟化技术。

## 安装Docker

安装好Docker后，可以执行两条命令查看Docker环境以及版本信息：

- `docker version`
- `docker info`



![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-01a57603583190cb7ae054f5afc1fe16_1440w.jpg)

## Docker镜像

如果说Docker容器本质上是一个运行的进程以及它需要的一些依赖，而Docker镜像则是定义容器的一个"模版"，容器则是镜像运行的一个实例。镜像是一个打包好的文件，里面包含了应用程序运行所需的所有库、配置和依赖。

## Docker镜像结构

所有的Docker镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的镜像层。以Dockerfile为例，每一行指令都产生一个新层。

镜像由一个或多个只读的镜像层构建而成，每个镜像层拥有独立的哈希值，Docker在拉取或推送镜像时，会判断哪几层在本地或远端已存在，避免不必要的操作。

## Docker镜像命令

我们可以通过下面这些命令进行一些Docker镜像相关的基本操作。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-e74d8ece473ee3de51365e527e32ba45_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5e0e7418b6097c775ac53ed90a478412_1440w.jpg)

## Docker容器

容器是镜像的一个运行的实例，用户可以从单个镜像上启动一个或多个容器。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-840510845f301dc7d15a983a2bd371c9_1440w.jpg)

## Docker容器的生命周期

容器在创建时进入Created状态，运行后进入Running状态，接着会进入到Pause状态或Exited状态，对已经退出的容器执行重启操作会使容器进入Restarting状态，随后转为Running状态。

## Docker容器操作

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-42ce3802d2915cb7663a1a869f93f9cf_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5bffeb705fe4c4249d34bfdfcd223681_1440w.jpg)

https://www.zhihu.com/video/1166455714251264000

创建一个Docker容器

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-f4ac2c4a46c00f66a24af7dbd7cda6f7_1440w.jpg)

## Dockerfile

Dockerfile是一个用以构建镜像的文本文件，它包括了一系列的执行用来表示当前应用的描述、依赖以及该如何运行这个应用，Docker Engine通过解析Dockerfile里的指令，来构建一个Docker镜像。

执行docker build命令会从顶层开始解析Dockerfile中的指令并逐行执行。而对每一条指令，Docker都会检查缓存中是否已经有与该指令对应的镜像层。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-53e9c534942ca4cf9890a29917e3f334_1440w.jpg)

https://www.zhihu.com/video/1166455944463732736

编写一个Dockerfile

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-f44259f9f250b8b864659e732e73636f_1440w.jpg)

## 多阶段构建

当Dockerfile中包含多个FROM语句时，触发多阶段构建。使用多阶段构建可以文件从另一个阶段复制到另一个阶段，生成的镜像将会是最后一个阶段构建的结果。

这个功能在某些情况下会十分有用，比如我们在编译应用时需要用到许多依赖，但想要保持镜像尽可能地小，这时候就可以通过多阶段构建来将编译环境和运行环境分离，在最终生成的镜像中只保留编译好的可执行文件。

https://www.zhihu.com/video/1166456122667143168

Dockerfile多阶段构建

## 应用容器化

这时候归纳一下应用容器化的过程：

1. 首先获取和编写应用代码
2. 为应用编写Dockerfile
3. 通过Dockerfile来构建应用的镜像
4. 将镜像交付到物理机和云端
5. 最终测试和运行容器化应用。

## Docker Registry

Docker Registry是镜像存储和分发的仓库，以完成镜像的搜索、拉取和推送等操作。

推送一个镜像：

- docker push ubuntu（[http://docker.io](https://link.zhihu.com/?target=http%3A//docker.io)）
- docker push myregistrydomain:port/foo/bar（myregistrydomain:port）

拉取一个镜像：

- docker pull ubuntu（[http://docker.io](https://link.zhihu.com/?target=http%3A//docker.io)）
- docker pull myregistrydomain:port/foo/bar（myregistrydomain:port）

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-8808eb070e6467d731028abab8b2b523_1440w.jpg)

## Docker数据管理

下面介绍一下Docker是怎么管理和存储数据的，Docker提供了3种方法将数据从宿主机挂载到容器：volumes，bind mounts和tmpfs mounts。其中volume适合在不同的容器中共享数据和数据迁移，bind mount适合宿主机与容器之间共享配置和代码，tmpfs适合不需要持久化数据的场景。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-10ad1b00527329f6106ad21e156a111a_1440w.jpg)

## Docker网络

Docker基于CNM实现了网络模型，CNM模型由三个部分组成：Sandbox，Endpoint，Network。

Docker内置了以下几种网络驱动，其中bridge桥接网络是Docker默认的网络驱动，在这种模式下，连接到同一个网桥的容器会使用同一个DNS解析服务，容器间可以使用这个解析服务来相互通信，而overlay是swarm模式下默认的网络驱动。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-a2fb956b5667a8afdc212e55b4c50b51_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5680b4b84c80472bdb9023fbc8389ba4_1440w.jpg)

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-45e784fd6922ec933374811f62c07b4c_1440w.jpg)

https://www.zhihu.com/video/1166460676783669248

Docker DNS

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3a0380d3c29b8c34ea04a2c4d2c1af23_1440w.jpg)

## Docker Machine

Docker Machine是一个用来在虚拟机安装Docker Engine的工具，以及通过命令行来管理和登录这些主机。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-250ed063fb326807967063a314d49aeb_1440w.jpg)

## Docker Compose

Docker Compose是一个定义和运行多容器应用的工具，通过编写Compose文件来定义和配置应用，只需要用docker-compose一条命令就可以一次把所有定义好的服务启动起来。

https://www.zhihu.com/video/1166457477158903808

Docker Compose部署应用



![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-e6877502c5843b97c6cfa4f850e6f14f_1440w.jpg)

## Docker Swarm

Docker Swarm是Docker原生的集群管理和调度工具，它把多个Docker主机抽象成单个虚拟的系统，开发者方便地可以通过Docker Swarm在多个主机上来调度和部署Docker容器。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-c3d9062349fdaced5e97e7e8422ff90f_1440w.jpg)

在Swarm中有几个概念，Node代表着Swarm集群中的一个Docker主机，其中每个主机可以是Manager或者Worker两个角色，他们分别代表管理节点和工作节点。Task可以理解为一个Docker容器以及其运行的命令，Service是一组Task的集合与定义，即运行在Node上的服务。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-459ac5eb2cfd2f39854fa4f71a34f860_1440w.jpg)

Docker Swarm提供了多种调度策略，来按需选择将容器分配到不同的节点上。

https://www.zhihu.com/video/1166463996525010944

使用Swarm部署应用

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-cee51446cd5698befab219bd78fdc9dd_1440w.jpg)

