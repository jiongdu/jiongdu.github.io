---
title: NVMe over RDMA浅析
date: 2017-04-05 20:49:59
tags: NVMe over RDMA
---
NVMe是一种Host与SSD之间的通信协议，为了把NVMe扩展到端到端的跨网络传输，NVMe的开发者提出了NVMe over Fabrics，用于解决将NVMe置于各种传输环境下所遇到的问题。  
<!--more--> 
### NVMe over Fabrics整体架构
NVMe over Fabrics的整体架构如下图所示。   
![](http://i.imgur.com/vvD1rC7.png)    
可以看出，NVMe Host(Controller)-side Transport Abstraction这两层便是NVMe over Fabrics协议的实现层。该协议只是一个用于NVMe Transport的抽象层而已，它并不实现真正的命令和数据传输功能，它只是为命令和数据传输定义了统一的规范，因此该协议只是“指导方针”。它是构建在"Fabrics"之上的，即它并不关心实际的Fabrics到底是什么，它只是提供了Fabrics通用的对接NVMe的接口，完成了对NVMe接口和命令在各种Fabrics而非只是PCIe上（NVMe Base协议只涉及PCIe这一种Fabric）的拓展。因此，为了使NVMe可以架构于不同的Fabric之上，各Fabric还需开发专用的功能实现层，真正实现基于此Fabric的数据传输功能，并完成和Transport Abstraction抽象层（即NVMe over Fabrics协议的实现层）的对接以使得传输抽象层可以调用到这些函数。         
### NVMe over RDMA
#### 整体架构
[RDMA技术浅析](http://blog.dujiong.net/2017/02/27/RDMA/)一文中介绍了RDMA这种"Fabric"以及它的几种实现方式。鉴于后续研究需要，这里选择RoCE（如无特别说明，均指的是v2版本）这种"Fabric"来展开说明。     
NVMe over RoCE的架构如下图所示。    
![](http://i.imgur.com/Ukm4No4.png)    
NVMe Host和NVMe Subsystem Controller是NVMe Base协议扩展到NVMe over Fabrics的部分；NVMe Host(Controller)-side Transport Abstraction则是NVMe over Fabrics传输抽象层的实现。RoCE层则是支持RoCE技术的网卡及相关驱动和RDMA协议栈，而不论InfiniBand、RoCE或者iWarp何种具体的RDMA实现形式，都约定提供统一的操作接口，RDMA Verbs便是RDMA技术向上层提供的接口。NVMe RDMA则是实现将RDMA的接口Verbs和NVMe对接的关键粘合层，简言之，其作用是将NVMe Transport Abstraction传输抽象层提供的传输接口可以调用到下层RDMA提供的传输接口（即verbs）。  
#### 具体实现    
此架构的具体操作原理将结合Linux 4.8版本内核的实现来解读。在4.8版本的Linux内核中加入了符合上图逻辑的NVMe over RDMA的代码。因此对于我们来说，只要编译最新内核并添加相关模块完成配置后，内核的代码完全不需要添加或修改。对于用户空间的应用程序来说，使用NVMe over RDMA的操作与传统的文件读写操作并无差异。这得益于Linux内核的软件分层架构，虚拟文件系统层、文件系统层和通用块层的架构可以屏蔽底层的硬件细节，且对上层提供统一的接口，这样上层用户空间的应用程序自然不会（也不应该）关注到底层硬件及使用协议的差别。Host端的Linux kernel的架构如下图所示。   
![](http://i.imgur.com/a9Nzq2g.png)   
如果用户使用NVMe特有的命令，即当用户下发NVMe commands时，这需要通过Linux内核的IOCTL接口，该接口的目的本就是实现用户和底层驱动打交道。可以看到，NVMe commands通过IOCTL机制便到达NVMe Common代码层，该层代码的任务就是解析并执行各种NVMe command。在NVMe Base标准的定义中，命令是被封装在capsule这一数据结构中传输，应用将capsule压入内存中的NVMe Submission Queue，接着controller取出capsule并解析应用的command，处理之后将回复capsule压入NVMe Completion Queue，应用取出回复消息完成一次通信。具体过程参考前文[NVMe技术浅析](http://blog.dujiong.net/2017/02/17/NVMe/)      
而在NVMe over RDMA的环境下，传输过程如下图所示。    
![](http://i.imgur.com/zqCUGcT.png)    
首先，host的command被封装进capsule后压入Host NVMe Submission Queue；接着该capsule会被放入Host RDMA Send Queue变成RDMA_SEND消息的消息负载；接着消息被host的RDMA网卡发包，当被target网卡接收后，capsule被放到target网卡的RDMA ReceIve Queue；接下来该capsule被放置到target端的内存中并由target处理该command；待处理完后，target生成回复信息（Respond Command）并封装进capsule然后将该回复capsule压入target端的NVMe Controller的Submission Queue，并经由target的网卡发送给host。这样，一次NVMe over RDMA的通信就完成了。
#### NVMe over RDMA读写文件原理    
下面以host读文件为例讨论数据流的传输。      
事实上，在NVMe over Fabrics协议中，提出了两种数据传输方式，一种是将data附到capsule中，只要在传输command时负载需要传输的data即可。另一种则是直接的内存传输方式。这种方式原理上是将要传输的数据的地址和长度等元信息负载到capsule中。如用户发出read请求后，最终的capsule消息负载中会包含数据的存储地址、要读取的数据的长度以及要读到的内存地址。利用RDMA技术，这种传输的特性将表现的更加淋漓尽致。     
由Linux内核知识可知，虚拟文件系统（VFS）向上层提供统一的文件操作接口，当用户空间进程读取文件时便调用read API；接着VFS调用到真正的文件系统（例如Ext4等）真正的read函数，真正文件系统的作用是确定数据的位置，简言之就是根据用户要读取的文件、长度、偏移量等确定文件的逻辑块号（在真实的文件存储中，一个文件是被分割成若干块存储的）；接下来内核利用通用块层（Block Layer）启动IO操作来传送所请求的数据，通用块层为所有的块设备提供了一个抽象视图，隐藏了硬件块设备间的差异，而每次IO操作由一个“块I/O”结构（struct bio）的对象来描述。至此，所有的读文件操作都是这套同样的流程。由于NVMe或AHCI都是更底层的接口标准，因此差异从通用块层之下才开始。       
此后，进入到NVMe Transport Abstraction（即NVMe over Fabrics协议的实现层）或者NVMe over PCIe，这时便会由此生成NVMe（base或over Fabrics）标准的Read Command，由NVMe Base标准对Read Command的定义，Read Command中包含了数据应该读到的内存区域的地址（Read Command的Data Pointer字段）、以及存储数据的逻辑块号的起始地址（Starting LBA字段）等其他在read操作中必需的字段（显然地，这些信息从上层通用块层传递的bio对象中获取），并将该Read Command封装进NVMe的通信单元capsule中。接着便进入到下一层的RDMA stack。RDMA Stack之下便是支持RDMA技术的网卡驱动，最底层便是不同技术（IB/RoCE/iWarp）实现的RDMA网卡。内核的NVMe RDMA层代码完成调用RDMA的接口verbs将该capsule封装为RDMA\_SEND消息的负载。              
Target的controller处理完read command后，数据流传输便开始了。数据流的传输则完全借助于RoCE技术。由于此前在target端注册了host的内存，接下来从target的SSD中读出的数据便直接封装为RDMA\_WRITE消息的负载（注意host的Read Command触发的却是target的RDMA\_WRITE操作），然后将这部分数据直接从target端写入到host的内存中，而写入的地址在Read Command中便已指明。该过程的实现得益于RDMA的数据零复制技术。      

说了这么多，通过一幅图来直观展示上述NVMe over RDMA读文件的过程。     
![](http://i.imgur.com/Mi5Okr5.png)

### 总结
本文分析了将NVMe扩展到端到端的跨网络传输---NVMe over RDMA的架构和实现原理。以读文件为例，详细分析了数据流的传输过程。后续将进入实战阶段，在服务器上搭建小型NVMe存储系统，并通过RDMA进行跨网络传输。




