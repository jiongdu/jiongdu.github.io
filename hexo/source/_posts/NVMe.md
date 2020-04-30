---
title: NVMe技术浅析
date: 2017-02-17 20:09:40
tags: NVMe over RDMA
---
在[存储的那些词儿](http://blog.dujiong.net/2017/02/13/storage/)一文中已经提到了NVMe是使用PCIe通道的一种逻辑设备接口标准（是接口标准，不是接口！），本文将进一步地分析NVMe的设计及其带来的性能提升。
<!--more-->
### NVMe设计
NVMe指定了Host与SSD之间通信的命令，以及命令执行的流程。NVMe由两种命令组成，一种是Admin Command，用于Host管理和控制SSD；另外一种是I/O Command，用于Host和SSD之间数据的传输。   
同ATA中定义的命令相比，NVMe的命令个数少了很多，完全就是为SSD量身定制的。这里放一张NVMe1.2支持的IO Command列表。    
![](http://i.imgur.com/INLzJpX.png)  
除了指定Host与SSD之间的通信命令以外，NVMe还定义了命令的执行流程。NVMe中定义了三个关键组件用于命令和数据的处理：Submission Queue（SQ）、Completion Queue（CQ）和Doorbell Register（DB）。SQ和CQ位于Host的内存中，DB位于SSD的控制器内部。在详细说明NVMe命令的处理流程之前，先上一张图宏观感受下PCIe系统和NVMe的结合。图中的NVMe Subsystem一般就是SSD，作为一个PCIe Endpoint通过PCIe连接着Root Complex（RC），然后RC连接着CPU和内存。          
![](http://i.imgur.com/vl7O3dS.png)    
下面通过SQ、CQ和DB这三个关键组件来总体认识NVMe是如何处理命令的。SQ位于内存中，Host要发送命令时，先把准备好的命令放在SQ中（每个命令条目大小是64字节），然后通知SSD来取；在通知中发挥作用的就是DB寄存器，Host通过写SSD端的DB寄存器来告知SSD执行命令；此外，CQ也是位于Host内存中，一个命令执行完成，成功或失败，SSD总会往CQ中写入命令完成状态（每个条目是16字节）。    
下图展示了NVMe一次完整处理指令的流程，共八步：    
（1）Host将指令写到SQ；    
（2）Host写DB，通知SSD取指令；   
（3）SSD收到通知，从SQ中取走指令；    
（4）SSD执行指令；    
（5）指令执行完成，SSD往CQ写入指令执行结果；    
（6）SSD通知Host指令执行完成；    
（7）Host收到通知，处理CQ，查看指令完成状态；     
（8）Host处理完CQ中的指令执行结果，通过DB回复SSD。   
![](http://i.imgur.com/Z25eGGp.png)   
#### SQ/CQ/DB详细分析
如前文所述，SQ/CQ/DB是NVMe处理命令的关键。Host往SQ中写入命令，SSD往CQ中写入命令完成结果。NVMe中有两种SQ和CQ，一种是Admin SQ，用于放Admin命令；一种是I/O SQ，放I/O命令。如下图所示，系统中只有一对Admin SQ/CQ，它们是一一对应的关系；I/O SQ/CQ却可以有很多，最大值65535（64K-1）。此外，Host端每个Core可以有一个或多个SQ，但只有一个CQ，即SQ与CQ可以是一对一或多对一的关系。这样设计的原因有二：一是性能需求，一个Core中有多线程，可以做到一个线程独享一个SQ；二是QoS需求，可以对多个SQ设置不同的优先级。  
![](http://i.imgur.com/bIIfhg0.png)   
此外，作为队列，每个SQ和CQ都有一定的深度，对Admin SQ/CQ来说，其深度可以是2-4096（4K），对I/O SQ/CQ，队列深度可以是2-65536（64K）。     
综上，NVMe中SQ/CQ的个数可以配置，每个SQ/CQ的深度也可以配置，即NVMe的性能是可以通过配置队列个数和队列深度来灵活调节的。这一点和AHCI相比是巨大的提升，因为我们知道，AHCI只有一个命令队列，且队列深度是固定的32。   
说了这么多SQ和CQ，DB呢？我们知道，SQ和CQ都是队列（且是环形队列），队列的头尾很重要，头决定谁会最先被服务，尾决定新到来命令的位置。所以，我们需要记录SQ和CQ的头尾位置，这就是DB的作用之一。DB是在SSD端的寄存器，记录SQ和CQ的头尾位置。每个SQ或CQ，都有两个对应的DB：Head DB和Tail DB。     
![](http://i.imgur.com/8IzJSna.png) 
从上图可以看出，这是一个生产者/消费者模型。对一个SQ来说，它的生产者是Host，因为它往SQ的Tail位置写入命令，消费者是SSD，它从SQ的Head取出指令执行。CQ则刚好相反，生产者是SSD，消费者是Host。   
DB的另一个作用是通知，Host更新SQ Tail DB的同时，也是在告知SSD有新的命令需要处理；Host更新CQ Head DB的同时，也是在告知SSD返回的命令完成状态信息已经被处理。     
下面以一个实例来说明SQ/CQ/DB三者配合的详细过程。    
（1）开始时假设SQ1和CQ1是空的，Head=Tail=0；        
![](http://i.imgur.com/L8WPeKE.png)    
（2）Host往SQ1中写入三个命令，SQ1的Tail变为3。Host在往SQ1写入三个命令后，同时去更新SSD Controller的SQ1 Tail DB寄存器，值为3。Host更新这个寄存器的同时，也是在告诉SSD Controller，有新命令了。
![](http://i.imgur.com/sRJyKnc.png)    
（3）SSD Controller收到通知后，于是派人去SQ1把3个命令都取回来执行。SSD把SQ1的三个命令都消费了，SQ1的Head从而也调整为3，SSD Controller会把这个Head值写入到本地的SQ1 Head DB寄存器。    
![](http://i.imgur.com/riHdErF.png)   
（4）SSD执行完了两个命令，于是往CQ1中写入两个命令完成信息，同时更新CQ1对应的Tail DB寄存器，值为2。同时，SSD发消息给Host告知有命令完成。       
![](http://i.imgur.com/UmkUyR6.png)      
（5）Host收到SSD的通知后，从CQ1中取出那两条完成信息处理。待处理完毕，Host往CQ1 Head DB寄存器中写入CQ1的head，值为2。    
![](http://i.imgur.com/vd75vaC.png)   
这样，就完成了一次完整的命令处理。    
### NVMe总结
NVMe所带来的重大改进主要包含以下几方面：一是低延迟，低延时和良好的并行性可以让SSD的随机性能大幅提升；其次是支持多队列和更高的队列深度，多队列让CPU的性能得到更好的释放，而队列深度从32提升到最大64K，则大幅提升了SSD的IOPS能力；然后是NVMe的低功耗，其加入了自动功耗状态切换和动态能耗管理功能；最后是其驱动的适用性广，解决了不同PCIe SSD之间的驱动适用性问题。支持NVMe标准的PCIe SSD可适用于多个不同平台，也不需要厂商独立提供驱动支持。目前Windows、Linux、Solaris、Unix、VMware、UEFI等都加入了对NVMe SSD的支持。

### 附
特别鸣谢：   
[ssdfans](http://www.ssdfans.com/)

