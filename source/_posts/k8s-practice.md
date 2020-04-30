---
title: k8s实践系列---从Docker到k8s
date: 2019-08-12 22:56:20
tags: k8s
---
在容器化和云原生技术大行其道的时代，特推出《k8s实践》系列文章，就云原生实践过程中的一些经验与大家分享。本文为第一篇：从Docker到k8s。

<!--more-->
[TOC]

## 传统模式存在的问题
传统基于虚拟机构建的服务发布、管控平台主要存在以下问题：
![传统模式的问题](http://km.oa.com/files/photos/pictures/201906/1561283957_74_w1208_h497.png)
1、发布繁琐：服务发布所涉及的代码编译-->应用打包-->版本升级的整个过程，都需要开发人员介入，缺乏一套自动化的流水线；   
2、人工扩缩容：在一些大型运营活动、突发事件等流量突增的场景下，需要开发及运维人员介入进行手动扩缩容，整个过程显得比较笨拙和滞后；    
3、故障无自愈：线上服务或节点出现故障时，平台无有效的自愈能力；   
4、机器裁撤：物理机裁撤时，不能做到优雅的迁移线上服务；    
5、资源利用率低：由于虚拟机对操作系统进行了虚拟化，这使得虚拟机的隔离性好，但资源利用率比较低；    
6、环境依赖：比如常见的gcc版本不匹配导致的代码编译不过，程序运行异常等问题；      
这些问题极大地降低了我们的开发和运维效率，甚至是影响线上服务质量，因此，我们迫切地需要更加智能、自动化的服务管控平台。

---

## Docker初出茅庐
### Docker的核心贡献
Docker本质上只是一个使用容器技术实现的“沙盒”而已，通过操作系统中的Namespace和Cgroups机制为每一个应用创建一个独立的“沙盒”，然后在“沙盒”中启动这些应用进程。但Docker能迅速崛起并占领容器市场的根本原因在于Docker镜像。在Docker之前，PaaS(Platform as a Service)上部署应用最麻烦的问题在于应用程序的打包，整个过程毫无章法可循，基本上靠不断试错。很多时候在本地运行正常的应用，却需要做很多修改和配置才能在PaaS里运行起来。      
Docker镜像解决的核心就是打包这个根本性问题。所谓Docker镜像，本质上就是一个压缩包，但是这个压缩包里的内容，比PaaS的可执行文件+启停脚本要丰富很多，实际上，大多数Docker镜像是直接由一个完整操作系统的所有文件和目录组成的，所以这个压缩包里的环境和本地开发、测试的环境是完全一样的。因此，Docker这种便利的打包机制，保证了服务本地环境和云端环境的高度一致，避免了用户通过不断试错来调试环境的痛苦过程。
### Docker依赖的核心技术
#### Namespace
Namespace是Linux提供的一种内核级别的资源隔离机制，是Linux创建新进程的一个可选参数，每个Namespace下的资源相对于其他Namespace都是透明、不可见的。目前Linux系统提供了IPC、PID、Mount、UTS、Network、User、Cgroup等七个资源隔离类型。  
![Namespace](http://km.oa.com/files/photos/pictures/201906/1561284030_18_w683_h385.png)
Namespace是Docker容器中用来实现“隔离”的技术手段，Namespace技术实际上是修改了进程看待整个计算机的“视图”，即是被操作系统限制了只能看到某些指定的内容，但对于宿主机来说这些被隔离的进程与其他进程是没有差别的。所以，docker的最基本原理实际上就是在创建容器进程时，指定了这个进程所需要的一组Namespace参数，这样容器就只能看到当前Namespace所限定的资源、文件、设备、配置等。因此，这也印证了docker容器的本质其实是一种特殊的进程。  
比如我们使用`docker exec -it 4ddf5638583c /bin/bash`命令进入到`4ddf5638583c`这一docker容器中，其原理是通过把`/bin/bash`进程加入到`4ddf5638583c`容器进程的Namespace中，这样我们在shell中看到的就是`4ddf5638583c`容器进程的视图。
#### Cgroups
既然Docker容器进程只是宿主机上的一个应用进程，那必然存在和宿主机上的其他进程竞争资源的问题。这个问题可以通过Linux Cgroups来解决。Linux Cgroups的全称是Linux Control Group。它最主要的作用是限制一个进程组能够使用的资源上限，包括CPU、内存、磁盘、网络带宽等等。   
在Linux中Cgroups暴露给用户的接口是文件系统，它以文件和目录的形式组织在操作系统的`/sys/fs/cgroup`路径下，使用`mount -t cgroup`可以查看Cgroups的挂载目录，同时可以看到`/sys/fs/cgroup`目录下有很多子目录如`cpu`、`cpuset`、`memory`等，这些表示当前可用被Cgroups限制的资源类型。   
下面通过一个简单的测试来了解Cgroups是如何限制进程资源使用的。
![Cgroups](http://km.oa.com/files/photos/pictures/201906/1561284053_34_w1476_h341.png)
1、在`/sys/fs/cgroup/cpu`目录下创建一个名为`mytest`的控制组；     
2、初始时`cpu.cfs_period_us=1`,代表CPU时间片使用没有上限，测试中的进程为shell命令`while : ; do : ; done &`，使用top命令发现当前进程直接把CPU跑到了100%；      
3、然后修改`cpu.cfs_quota_us`文件，让进程只能占用30%的CPU时间片`echo 30000 > cpu.cfs_quota_us`（默认`cpu.cfs_period_us`值为100000表示100ms，设置30000表示30ms占比30%）；
![](http://km.oa.com/files/photos/pictures/201906/1561284075_47_w839_h84.png)
4、再次使用top命令查看，发现CPU利用率为30%。       
#### rootfs
为了让容器的根目录看起来更真实，我们一般会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如Ubuntu 16.04的镜像。这样在容器启动之后，我们在容器里通过执行`“ls /”`查看根目录下的内容，就是Ubuntu 16.04的所有目录和文件。          
这个挂载在容器根目录、用来为容器进程提供隔离后执行环境的文件系统，就是“容器镜像”，也被称为“rootfs（根文件系统）”。rootfs只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在Linux系统中，这两部分是分开存放的，所以容器镜像其实是打包了操作系统相关的一些文件，并且会对一些不必要的内容进行裁剪，所以docker镜像本身是不大的。此外，由于rootfs里打包的不只是服务，而是整个操作系统的文件和目录，这意味着服务以及它运行所需的环境依赖都被打包在一起，从而解决了环境依赖问题。       
在rootfs的基础上，Docker的镜像设计创造性地提出了使用多个增量rootfs联合挂载一个完整rootfs的方案，即镜像层的概念。用户制作镜像的每一步操作，都会生成一个层，也就是一个增量rootfs，然后通过联合文件系统（Union File System）将多个不同位置的目录联合挂载到同一个目录。
![](http://km.oa.com/files/photos/pictures/201906/1561355016_97_w659_h333.png)     
一个容器的rootfs主要由三部分组成：            
1、只读层：如上图中rootfs最下面的四层，可以看到，它们的挂载方式都是只读的（`readonly+whiteout`）。         
2、可读写层：如上图中rootfs最上面的一层，它的挂载方式为`read write`，在没有写入文件之前，这个目录是空的，而一旦在容器里做了写操作，我们修改所产生的文件就会已增量的方式出现在可读写层中。此外，为了实现对可读层中文件的删除操作，Union FS会在可读写层创建一个whiteout文件，把只读层里的文件“遮挡”起来，这样当两个层被联合挂载的时候，只读层的文件就被“删除”了。          
3、Init层是Docker项目单独生成的一个内部层，夹在只读层和可读写层之间，专门用来存放/etc/hosts、/etc/resolv.conf等信息。因为用户往往需要在启动容器时写入一些指定的值如hostname，所以需要在可读写层对它们进行修改，但是这些修改往往只对当前容器有效，我们不希望执行docker commit时这些信息连同可读写层一起被提交。因此Docker的做法是在修改了这些文件之后，以一个单独的层挂载出来。         

### Docker小结
总的来说，一个docker容器，实际上是由一个Linux Namespace、Linux Cgroups和rootfs三种技术构建出来的进程隔离环境。更直观地讲，一个正在运行的Linux容器，可以被“一分为二”地看待：      
1. 一组联合挂载在`/var/lib/docker/aufs/mnt`上的rootfs，这一部分我们成为docker镜像，是docker容器的静态视图；
2. 一个由Namespace+Cgroups构成的隔离环境，这一部分我们称为“容器运行时”（Container Runtime），是容器的动态视图。                 

---

## 从Docker到Kubernetes
### 容器编排与部署问题
通过上述对Docker的介绍不难发现，Docker对于个人开发者开发“单机版”容器化应用来说，确实已经足够好用了，无论是资源利用率还是开发效率都大为改善。但是如果在企业级应用场景中，很快就会因为容器规模的增加导致Docker手动操作方式的弊端不断被放大。因此，如何批量创建、调度和管理容器就成了制约Docker技术在企业级应用场景的主要障碍，即容器的编排与部署问题。
### Kubernetes简介及优势分析
Kubernetes（简称k8s）是由Google开源的容器编排工具，是Google大规模容器管理系统Borg的开源版本实现，吸收借鉴了Google过去十年间在生产环境上所学到的经验与教训。Docker开发公司在容器编排上提供了Swarm解决方案，但k8s更灵活，功能更强大，社区发展更活跃，已经成为企业级容器编排工具的首选。
![k8s](http://km.oa.com/files/photos/pictures/201906/1561284411_46_w342_h312.png)
简单来说，k8s是一个管理跨主机容器化应用的系统，实现了包括编排、部署、高可用管理和弹性伸缩在内的一系列基础功能并将其封装为一套简单易用的RESTful API（或是基于RESTful API封装的命令行工具）对外提供服务。k8s通过各种调度器和控制器进程来维护容器集群一直处于用户所期望的运行状态，并基于此建立了一套完整的集群自愈机制，包括容器的自动重启、自动调度和自动备份等。                     
但是，以上的这些功能不是k8s的核心竞争力，因为类似于Fleet或者Mesos这样的平台也提供了相差无几的能力。k8s着重瞄准并解决的问题是运行在大规模集群中的各任务之间的复杂关系处理，而这也被认为是作业编排和管理系统最困难的地方。因此，k8s项目从一开始就没有像同时期的各种“容器云”项目那样，把Docker作为整个架构的核心，而仅仅把它作为最底层的一个容器运行时实现，k8s主要针对的服务对象是由多个容器组合而成的复杂应用，如弹性、分布式的服务架构，为此，k8s引入了专门队容器进行分组管理的Pod，从而建立了一套将非容器化应用平滑地迁入到容器云上的机制。可以说，整个k8s的设计思想都是建立在Pod这个可以视作单个容器的“容器组”展开的，当容器组代替容器成为系统的部署和调度粒度时，所构建出来的应用组织方式就和Fleet和Mesos有很大的区别。                  
因此，k8s最核心的设计思想和优势是，以统一的方式来定义和处理各任务之间的复杂关系，并且为将来支持更多种类的关系留有余地。            

### 在Kubernetes上部署第一个应用
最后，我们通过一个简单的示例来看看如何在k8s集群上部署应用。本文不对k8s集群的搭建做阐述，假设应用部署时已有自建或云厂商提供的容器云平台。在示例中，我们希望k8s来完成Nginx容器镜像的部署和管理，此外，我们还需要平台帮我们运行两个完全相同的Nginx容器副本，以负载均衡的方式共同对外提供服务。           
下面的yaml配置文件定义了一个Deployment（标准无状态应用，帮助我们创建和管理完全相同的各实例Pod，本系列后续文章会详述），其中包含两个Nginx Pod：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2				# 副本数为2
  selector:
    matchLabels:
      app: nginx-server    # 这个 Deployment 管理着那些拥有这个标签的 Pod 
  template:
    metadata:
      labels:
        app: nginx-server  # 为所有 Pod 都打上这个标签
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9	# nginx镜像
        ports:
        - containerPort: 80  
```

接着创建这个Deployment：    

```
$ kubectl apply -f nginx-deployment.yaml
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            2           5s
```
这样，两个完全相同的Nginx容器副本就被启动了，可以看到，这个Deployment管理着两个Nginx Pod：    

```
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-2558053219-0g590   1/1       Running   0          2m
nginx-deployment-2558053219-220xk   1/1       Running   0          2m
```
从此，我们可以通过管理这个Deployment来实现多副本、服务高可用、版本更新与回滚、自动扩缩容等等能力。

---

## 总结
本文为《k8s实践》系列文章第一篇：从Docker到k8s，主要介绍了传统基于虚拟机构建的服务发布和管控平台存在的问题、Docker的核心贡献和关键技术以及从Docker到Kubernetes的必然演进。         

