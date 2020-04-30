---
title: Linux网络协议栈数据处理流程---收包
date: 2017-04-13 14:15:59
tags: network/linux
---
本文详细记录了从网卡驱动加载和初始化开始，一直到网络数据包到达，最后驱动程序将其递交到网络协议栈的整个处理过程。
<!--more-->
下文中出现的代码均基于Linux内核3.13.0版本，网卡驱动igb。
### 概述
从整体上看，Linux网络协议栈对数据包的处理流程如下：    
1. 网卡驱动的加载和初始化   
2. 数据包到达网卡   
3. 网卡将数据包通过DMA的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化    
4. 网卡通过硬件中断的方式通知CPU有数据到来        
5. 在中断处理程序中，驱动程序调用NAPI进入软中断处理                        
6. 软中断进程ksoftirqd通过调用NAPI函数poll循环接收数据包，软中断进程运行在系统的每个CPU上。        
7. 数据被传递到skb buffer中，随后递交网络协议栈       
8. 如果RPS（Receive Packet Steering）被打开或者NIC有多个接收队列，接收到的数据包将分散在多个CPU中，这里涉及到多CPU的负载均衡     
9. 数据帧从队列中递交到网络协议栈   
10. 协议栈处理数据   
11. 数据被添加到socket缓冲区   

下面将详细描述每一步所经过的处理。
### 网卡驱动的加载和初始化
网卡驱动程序向内核注册一个初始化函数，当驱动加载的时候，该函数被内核调用。Linux内核中通过module\_init宏进行注册。       
网卡需要有驱动才能工作，驱动是加载到内核中的模块，负责衔接网卡和内核。驱动在加载的时候将自己注册进网络模块，当相应的网卡收到数据包时，网络模块会调用相应的驱动程序处理数据。   
Linux内核中igb初始化函数(igb\_init\_module)和驱动注册函数(module_init）[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L676-L697)，可以看出，igb\_init\_module函数的大部分工作是通过pci\_register\_driver来完成的。
![](http://i.imgur.com/DBOCp1s.png)     
   
#### PCI设备列表
我们知道，一个驱动程序可以支持一个或多个设备，而一个设备只会绑定给一个驱动程序。因此，驱动程序将其支持的所有设备保存在一个列表（PCI device IDs）中，内核使用这些表来决定加载驱动的类型。     
Linux内核中igb驱动程序所支持的PCI设备列表[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L79-L117)

	static DEFINE_PCI_DEVICE_TABLE(igb_pci_tbl) = {
		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I354_BACKPLANE_1GBPS) },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I354_SGMII) },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I354_BACKPLANE_2_5GBPS) },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I211_COPPER), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_COPPER), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_FIBER), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_SERDES), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_SGMII), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_COPPER_FLASHLESS), board_82575 },
  		{ PCI_VDEVICE(INTEL, E1000_DEV_ID_I210_SERDES_FLASHLESS), board_82575 },
		/* ... */
	};
	
	MODULE_DEVICE_TABLE(pci, igb_pci_tbl);

如前文所述，igb初始化函数igb\_init\_module的大部分工作由pci\_register\_driver完成。后者通过一系列函数指针的赋值以及PCI设备列表的注册完成了PCI网卡设备的初始化。

	static struct pci_driver igb_driver = {
		.name     = igb_driver_name,
		.id_table = igb_pci_tbl,
		.probe    = igb_probe,
		.remove   = igb_remove,
		/* ... */		
	};
	
#### PCI probe
一旦PCI设备通过其PCI ID被内核识别，内核就可以选择合适的驱动来控制该设备。这个过程是通过PCI驱动程序向内核PCI子系统注册的探测函数（probe function）完成的。其处理流程包括：       
1. 打开PCI设备       
2. 申请内存和I/O端口     
3. 设置DMA参数    
4. 注册驱动支持的ethtool调用函数              
5. struct net\_device\_ops结构的创建、初始化和注册。该结构体包含了指向打开设备、发送数据、设置MAC地址等操作函数的函数指针。   
6. struct net\_device结构体的创建、初始化和注册。sturct net\_device代表一个网络设备。 

下面是ibg驱动程序中igb\_probe函数的部分代码。[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L2016-L2429)   

	err = pci_enable_device_mem(pdev)
	/* ... */
	err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
  	/* ... */
	err = pci_request_selected_regions(pdev, pci_select_bars(pdev, IORESOURCE_MEM), igb_driver_name)；
	pci_enable_pcie_error_report(pdev);
	
	pci_set_master(pdev);
	pci_save_state(pdev);
	/* ... */
	netdev->netdev_ops = &igb_netdev_ops;

以及net\_device\_ops结构体中函数指针的初始化。[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L1905-L1913)

	struct const struct net_device_ops igb_netdev_ops = {
		.ndo_open = igb_open,
		.ndo_stop = igb_close,
		.ndo_start_xmit = igb_xmit_frame,
		.ndo_get_stats64 = igb_get_stats64,
		.ndo_set_mac_address = igb_set_mac,
		.ndo_change_mtu = igb_change_mtu,
		.ndo_do_ioctl = igb_ioctl,
		
		/* ... */



下面说明驱动中注册ethtool调用函数的过程。     
ethtool是一个命令行程序，可以使用它来获取并配置网卡硬件和驱动。在ubuntu系统下，可以使用`apt-get install ethtool`来安装ethtool。    
ethtool最常见的用法是从网络设备中获取详细的统计数据。    
ethtool通过使用ioctl系统调用来和网卡驱动打交道。网卡驱动通过注册一系列函数来运行ethtool的操作指令。当ethtool发起了一个ioctl系统调用，内核将会在合适的驱动中寻找注册的ethtool结构并执行注册的函数。     
igb驱动程序仍然在igb\_probe完成ethtool操作的注册。    

	static int igb_probe(stauct pci_dev* pdev, const struct pci_device_id *ent){
		/* ... */
		igb_set_ethtool_ops(netdev);
	}
	
igb驱动中的ethtool操作的函数指针结构体及设置在 [源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_ethtool.c#L2970-L3015)

	static const struct ethtool_ops igb_ethtool_ops = {
		.get_settings = igb_get_settings,
		.set_settings = igb_set_settings,
		.get_drvinfo = igb_get_drvinfo,
		.get_regs_len = igb_get_regs_len,
		.get_regs = igb_get_regs,
		/* ... */
	}

	static int igb_probe(struct pci_dev *pdev, const struct pci_devie_id *ent){
		/* ... */
		igb_set_ethtool_ops(netdev);
	}

此外，需要说明的是，ethtool所实现的函数由驱动自身决定，因此，并不是所有的驱动都实现了ethtool的所有函数。

#### 更多关于Linux PCI/PCIe驱动的信息
PCI、PCIe设备驱动程序的细节可以参考[wiki](http://wiki.osdev.org/PCI)、[text file in the linux kernel](https://github.com/torvalds/linux/blob/v3.13/Documentation/PCI/pci.txt)等相关资料，这里不做更深层次讲解。      

### 中断和NAPI
当数据通过DMA的方式被传送到RAM之后，NIC需要一种通知机制来通告数据的到来。传统的方式是NIC产生一个硬件中断给CPU。但是，这种方法的缺陷在于，如果有大量的数据包到达，将会产生很多次的硬件中断，极大的降低了CPU处理的效率。      
在这种情况下，NAPI（New Api）被提出，用于减少（不是消除）数据包到达时硬中断的产生次数，下面将详细介绍NAPI机制。     
#### NAPI机制
NAPI的核心概念是不采用中断的方式读取数据，而是首先采用中断唤醒数据接收的服务程序，然后使用poll方法来轮询数据。这样可以防止高频率的中断影响系统的整体运行效率。    
当然，NAPI也存在一些缺陷，比如，对于上层的应用程序而言，系统不能在每个数据包接收到的时候都可以及时地区处理它，而且随着传输速度增加，累计的数据包将会耗费大量的内存；对大的数据包处理比较困难，原因是大的数据包传送到网络层上的时候耗费的时间比短数据包长很多。        
NAPI机制的大致处理流程如下：  
![](http://i.imgur.com/sPbd3RI.jpg)          
1. NAPI被驱动使能，但是初始时处于OFF状态    
2. 数据包到来并通过DMA方式传送到内存    
3. NIC产生硬中断，触发中断处理函数      
4. 中断处理函数中，驱动通过软中断唤醒NAPI子系统，通过在一个单独执行线程上调用注册的poll函数接收数据包    
5. 网卡驱动程序关掉硬中断，从而允许NAPI子系统在处理数据包期间不会被中断。    
6. 一旦处理完成，NAPI子系统被关闭，网卡硬件中断打开。        

#### igb驱动程序实现
下面结合igb驱动程序详细说明。     
1.注册poll函数      
设备驱动程序实现了poll函数，并通过调用netif\_napi\_add函数将其注册到NAPI子系统。该过程发生在驱动初始化的过程中。igb驱动中注册的相关代码[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L1145-L1271)   

	static int igb_alloc_q_vector(struct igb_adapter* adapter, int v_count, int v_idx, int txr_count, int trx_idx, int rxr_count, int rxr_idx){
		q_vector = kzalloc(size, GFP_KERNEL);	
		if(!q_vector)
			return -ENOMEM;
		netif_napi_add(adapter->netdev, &q_vector->napi, igb_poll, 64);
	/* ... */
	}   
   
2.打开网络设备     
当网络设备被使能（比如，ifconfig eth0 up），net\_device\_ops中的ndo\_open所指向的函数(igb驱动中的igb\_open)将会被调用。完成以下处理：    
（1）分配RX和TX队列的内存空间；     
（2）打开NAPI     
（3）注册中断处理函数     
（4）打开硬中断   
3.网卡环形缓冲区和多队列       
现在，大多数网卡都采用基于环形缓冲区的队列来进行DMA的传送。因此，驱动程序需要同操作系统配合为NIC预留一块可以使用的内存区域。一旦该内存区域被分配，硬件将会被通知其地址并且到达的数据将会被写入到RAM。     
但是，如果数据包接收速率很高以致于一个CPU不能处理呢？ 这种情况下，就会导致大量丢包的发生。所以，RSS(Receive Side Scaling)和网卡多队列技术被提出。     
RSS是一种能够在多处理器系统下使接收报文在多个CPU之间高效分发的网卡驱动技术。主要思想是基于Hash来实现动态的负载均衡，关于RSS，后面还会详细介绍。             
而网卡多队列是指有些网卡能够同时将数据包写入到不同的区域，每一个区域都是一个单独的队列。这种情况下，操作系统可以使用多CPU在硬件层面上并行的处理到来的数据。      
Inter I350 NIC支持多队列，可以在igb驱动程序中找到对应函数([源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L2801-L2804))   
	
		err = igb_setup_all_rx_resources(adapter);    
		if(err)     
			goto err_setup_rx;

igb\_setup\_all\_rx\_resources函数调用igb\_setup\_rx\_resources（[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L3098-L3122)）来为每一个RX队列DMA memory来存放网卡收到的数据。    
   
	static int igb_setup_rx_resources(struct igb_adapter* adapter){
		struct pci_dev* pdev = adapter->pdev;
		int i, err = 0;
		for(i=0;i<adapter->num_rx_queues;i++){
			err = igb_setup_rx_resources(adapter->rx_ring[i]);
			/* ... */
		}
		return err;
	}

4.Enable NAPI     
前面我们已经看到驱动程序如何将poll函数注册到NAPI子系统，但是，NAPI通常会等到设备被打开之后才会开始工作。            
使能NAPI比较简单。在igb驱动中，调用napi\_enable实现。  
  
	for(i=0;i<adapter->num_q_vectors;i++){
		napi_enable(&(adapter->q_vector[i]->napi));
	}

5.注册中断处理函数      
使能NAPI后，接下来需要注册中断处理函数。通常情况下，设备可以采用不同的方式来通告中断: MSI-X, MSI和传统的中断方法。其中，MSI-X中断是较好的方法，特别是对于支持多RX队列的NIC，每个RX队列都有其特定分配的硬中断，可以被特定的CPU处理。关于这三种具体的中断机制，这里不再深究。       
驱动程序必须采用device支持的中断方法注册合适的处理函数，以便当中断发生时，可以正常调用。       
在igb驱动中，函数igb\_msix\_ring，igb\_intr\_msi，igb\_intr分别是对应于MSI-X,MSI和legacy模式的中断处理函数。驱动程序将按照MSI-X->MSI和legacy的顺序尝试注册中断处理函数。[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L1360-L1413)
	
	static int igb_request_irq(struct igb_adapter* adapter){
		struct net_device* netdev = adapter->netdev;
		struct pci_dev* pdev = adapter->pdev;
		int err = 0;
		if(adapter->msix_entries){
			err = igb_request_msix(adapter);
			if(!err)
				goto request_done;
			/* fall back to MSI */
		}
		/* ... */
		if(adapter->flags & IGB_FLAG_HAS_MSI){
			err = request_irq(pdev->irq, igb_intr_msi, 0, netdev->name, adapter);
			if(!err)
				goto request_done;
			/* fall back to MSI */
		}
		/* ... */
		err = request_irq(pdev->irq, igb_intr, IRQF_SHARED, netdev->name, adapter);
		/* ... */
	}

以上便是igb驱动注册函数的过程，当NIC产生一个硬件中断信号表明数据包到来等待被处理时，这些函数将会执行。          
6.Enable Interrupts
到这里为止，几乎所有的设置已经完成，除了打开NIC中断，等待数据包的到来。打开中断是一个硬件操作，igb驱动通过函数igb\_irq\_enable写寄存器实现。[源码地址](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L1462-L1488)
	
	static void igb_irq_enable(struct igb_adapter* adapter){
		/* ... */
		wr32(E1000_IMS, IMS_ENABLE_MASK | E1000_IMS_DRSTA);
		wr32(E1000_IAM, IMS_ENABLE_MASK | E1000_IMS_DRSTA);
		/* ... */
	}

### 接收数据包处理
#### softirq  
先来看下Linux内核中的软中断。      
Linux内核中的软中断是一种在驱动层面上实现的在进程上下文之外执行中断处理函数的机制。该机制十分重要，因为硬件中断在中断处理程序执行期间可能会被关闭。关闭的时间越长，事件被错过处理的机会就越大。因此，在中断处理程序之外推迟长时间的事件处理十分重要，这样可以尽快完成中断处理程序并且重新打开设备中断。      
软中断可以被想象成一系列的内核线程（每个CPU一个） ，这些内核线程执行针对不同软中断事件注册的事件处理函数。比如，使用top命令，在内核线程列表中看到的ksoftirqd/0，就是运行在CPU 0上的。   
内核通过执行open\_softirq函数来注册软中断处理函数。    

#### ksoftirq
软中断对于延迟设备驱动的事件处理非常重要，因此ksoftirq进程随着内核启动很早就会运行。[相关源码](https://github.com/torvalds/linux/blob/v3.13/kernel/softirq.c#L743-L758)     

	static struct smp_hotplug_thread_softirq_threads = {
		.store             = &ksoftirq,
		.thread_should_run = ksoftirq_should_run,
		.thread_fn         = run_ksoftirqd,
		.thread_comm       = "ksoftirqd/%u". 
	};
	
	static __init int spawn_ksoftirqd(void) {
		register_cpu_notifier(&cpu_nfb);
		BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
		
		return 0;
	}  
	early_initcall(spawn_ksoftirqd);

#### Linux网络子系统
到现在为止，我们已经分析了网络驱动和软中断的工作流程，接下来开始分析Linux netdevice子系统的初始化和网络数据包到达之后的处理流程。   
netdevice子系统通过net\_dev\_init函数进行初始化。该函数将会对每个CPU创建struct softnet\_data结构体，该结构体包含了指向（1）注册到该CPU的NAPI结构体列表；（2）数据处理的backlog值；（3）RPS（Receive packet steering）值等重要数据的指针。    
此外，net\_dev\_init注册了两个软中断处理函数，分别用于处理到来和发送的数据包。

	static int __init net_dev_init(void){
		/* ... */
		open_softirq(NET_TX_SOFTIRQ, net_tc_action);
		open_softirq(NET_RX_SOFTIRQ, net_rx_action);
		/* ... */		
	}

后文中，我们很快就会看到驱动处理函数怎么“触发”注册在NET\_RX\_SOFTIQ（NET\_TX\_SOFTIQ）上的net\_rx\_action(net\_tx\_action)函数。   
#### 数据包到来
终于，数据包到来了。      
数据包到达RX队列之后，将通过DMA传输到RAM，并触发硬件中断。如前文所述，中断处理函数将尽量推迟更多的处理发生在中断上下文之外。因为在处理中断的过程中，其他中断可能会被阻塞。         
下面是igb驱动程序中MSI-X中断处理函数的[源代码](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L5148-L5158)，从中我们可以看出，中断处理程序的代码逻辑非常简单，调用igb\_write\_itr和napi\_schedule两个函数完成快速处理。前者用于更新特定于硬件的寄存器。后者唤醒NAPI处理循环，用于在软中断中处理数据包。       

	static irqreturn_t igb_msix_ring(int irq, void* data){
		struct igb_q_vector* q_vector = data;
		/* Write the ITR value calculated from the previous interrupt */
		igb_write_itr(q_vector);
		napi_schedule(&q_vector->napi);
		return IRQ_HANDLED;
	}

napi\_schedule是\_\_napi\_schedule函数的封装。后者主要完成了两个处理，一个是struct napi\_struct添加到当前CPU所关联的结构体softnet\_data中的poll\_list链表中，另一个则是调用\_\_raise\_softirq\_irqoff来触发NET\_RX\_SOFTIRQ软中断，从而执行在netdevice子系统初始化期间注册的net\_rx\_action函数。后文我们会看到，软中断处理函数nx\_rx\_action将会调用NAPI的poll函数来接收数据包。     

	void __napi_schedule(struct napi_struct *n){
		unsigned long flags;
		
		local_irq_save(flags);
		__napi_schedule(&__get_cpu_var(softnet_data), n);
		local_irq_restore(flags);
	}
	EXPORT_SYMBOL(__napi_schedule);

	static inline void __napi_schedule(struct softnet_data *sd, struct napi_struct* napi){
		list_add_tail(&napi->poll_list, &sd->poll_list);
		__raise_softirq_irqoff(NET_RX_SOFTIRQ);
	}    

最后，需要说明的是，中断处理程序做快速操作和将数据包接收处理推迟到软中断都是在同一CPU上进行的。这也是通常将中断处理绑定到固定CPU的原因。  

#### 多队列
如果NIC支持RSS、multi queue，或者想更好的利用好数据的局部性原理，我们可以设置使用一些特定的CPU来处理NIC中断。   
在决定调整IRQ处理的参数之前，应该首先检查是否已经在后台执行irqbalance，该进程将会尝试自动平衡IRQ和CPU之间的关系，并且会覆盖用户的设置。然后在/proc/interrupts中获取NIC中每个RX队列的IRQ numbers之后，就可以通过修改/proc/irq/IRQ\_NUMBER/smp\_affinity来调整CPU和IRQ的对应关系。

#### 数据包处理
一旦软中断softirq开始被处理并调用net\_rx\_action，数据包处理流程就正式开始了。        
net\_rx\_action遍历当前CPU队列中的NAPI列表，取出队列中的NAPI结构，并依次对其进行操作。正如前面提到的，net\_rx\_action将会调用NAPI的poll函数来接收数据包。     
igb驱动中net\_rx\_action的部分实现[代码](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c)如下：

	/* initialize NAPI */
	netif_napi_add(adapter->netdev, &q_vector->napi, igb_poll, 64);		//64: weight

	/* net_rx_action */
	wight = n->weight;
	work = 0;
 	if(test_bit(NAPI_STATE_SCHED, &n->state)){
		work = n->poll(n, weight);
		trace_napi_poll(n);
	}
	
	WARN_ON_ONCE(work->weight);
	budget-=work;

其中,weight代表RX队列的处理权重，budget表示一种惩罚措施，用于多CPU多队列之间的公平性调度。      
接收数据包的igb\_poll函数的部分实现[代码](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L5987-L6018)

	static int igb_poll(struct napi_struct* napi, int budget){
		struct igb_q_vector* q_vector = container_of(napi,struct igb_q_vector,napi);
		bool clean_complete = true;

		#ifdef CONFIG_IGB_DCA
        if (q_vector->adapter->flags & IGB_FLAG_DCA_ENABLED)
                igb_update_dca(q_vector);
		#endif
		/* ... */
		if (q_vector->rx.ring)
                clean_complete &= igb_clean_rx_irq(q_vector, budget);
		 /* If all work not completed, return budget and keep polling */
        if (!clean_complete)
                return budget;
		/* If not enough Rx work done, exit the polling mode */
        napi_complete(napi);
        igb_ring_irq_enable(q_vector);
		
		return 0;
	}
其流程主要包括：   
1.如果内核支持DCA（Direct Cache Access），CPU缓存将会被很好的命中；    
2.调用igb\_clean\_rx\_irq循环处理数据包，直到处理完毕或者达到budget，即一次调度时间完成。该循环中将会完成以下处理：    
（1）当可用缓冲buffer的时候，为到来的数据包分配buffer；    
（2）从RX队列中取buffer数据并存储在skb结构中；    
（3）检查取出的buffer数据是否是“End of Packet”，如果是，递交下一步处理；    
（4）验证数据头部等信息是否正确；    
（5）构建好的skb通过napi\_gro\_receive函数调用被递交到网络协议栈；   
3.检查clean\_complete标志位判定是否所有的工作已经完成，如果是，返回budget值，否则，调用napi\_complete关闭NAPI，并通过igb\_ring\_irq\_enable重新使能中断，以保证下次中断到来会重新使能NAPI。

#### Generic Receive Offloading(GRO)
GRO是硬件优化方法LRO（Large Receive Offloading）的软件实现。
两种方法的主要思想都是通过合并“类似”的数据包来减少传递给网络协议栈的数据包数量，达到降低CPU利用率的目的。       
这类优化方法可能带来的问题是信息的丢失。如果一个数据包含有一些重要的选项或标志位，这些选项或标志位可能在它和其他数据包合并的时候丢失。这也是通常不推荐使用LRO的原因，而GRO针对数据包合并的规则比较宽松。      
GRO作为LRO的软件实现，在数据包合并的规则上显得更加严格。    
顺便说一句，如果在使用tcpdump抓包时，看到一些很大的数据包，很有可能是系统启用了GRO。   
比如，TCP协议需要决定是否或者什么时候将ACK合并到已存在的数据包中。   
一旦dev\_gro\_receive执行完成，napi\_skb\_finish被调用，该函数或者释放不再需要的数据结构，因为数据包已经被合并；或者调用netif\_receive\_skb将数据传输到网络协议栈。   
napi\_gro\_receive将通过GRO完成网络数据的处理，并将数据传送到网络协议栈。其中大部分的处理逻辑通过调用函数dev\_gro\_receive来完成。[相关源码](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L3826-L3916)     
当dev\_gro\_receive执行完成后，napi\_skb\_finish函数被调用，完成多余数据结构的清理（因为数据包已经被合并），或者调用netif\_receive\_skb将数据传送给网络协议栈。 
#### RSS和RPS     
在讲解netif\_receive\_skb之前，我们先来了解下Receive Packet Steering（RPS）和Receive Slide Steering（RSS）这两种用于多CPU负载均衡的处理机制。    

RSS用于多队列网卡，能够将网络流量分载到多个CPU上，降低单个CPU的占有率。默认情况下，每个CPU核对应一个RSS队列，驱动程序将收到的数据包的源、目的IP和端口号等，交由网卡硬件计算出一个hash值，再根据这个hash值来决定将数据包分配到哪个队列中。       
可以看出，RSS是和硬件相关联的，必须要有网卡的硬件进行支持。          
RPS可以认为是RPS的软件实现。RPS主要是把软中断负载均衡到各个CPU。简单地说，是网卡驱动对每个流生成一个hash标识，然后由中断处理程序根据这个hash表示将流分配到相应的CPU上，这样就可以比较充分地发挥多核的能力。   
可以看出，RPS是在软件层面模拟实现硬件的多队列网卡功能，主要针对单队列网卡多CPU环境，如果网卡本身支持多队列的话RPS就不会有任何的功能。
    
因此，netif\_receive\_skb将根据是否设置RPS对数据包进行不同的操作：        
（1）no RPS      
如果没有配置RPS，netif\_receive\_skb将会调用\_\_netif\_receive\_skb，后者在做一些信息的记录之后，调用\_\_netif\_receive\_skb\_core将数据移动到协议栈。    
（2）RPS    
如果RPS被打开，netif\_receive\_skb将会调用get\_rps\_cpu来计算hash并决定使用哪个CPU的积压队列。[源码地址](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L3699-L3705)，具体的入队操作则由get\_rps\_cpu调用enqueue\_to\_backlog完成。   

	cpu = get\_rps\_cpu(skb->dev, &rflows);
	
	if(cpu>=0){
		ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
		rcu_read_unlock();
		return ret;	
	}
      
enqueue\_to\_backlog首先得到指向CPU的softnet\_data结构体的指针，该结构体中含有指向input\_pkt\_queue队列结构的指针。因此，enqueue\_to\_backlog函数首先检查input\_pkt\_queue队列长度[源码](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L3199-L3200),只有当队列长度同时不超过netdev\_max\_backlog和flow limit的值，数据包才会入队，否则将会被丢弃。    
	
	qlen = skb_queue_len(&sd->input_pkt_queue);
	if(qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)){
		if(skb_queue_len(&sd->input_pkt_queue)){
	enqueue:
			__skb_queue_tail(&sd->input_pkt_queue, skb);
			input_queue_tail_incr_save(sd, qtail);
			rps_unlock(sd);
			local_irq_restore(flags);
			return NET_RX_SUCCESS;
		}
	}

同没有配置RPS时一样，backlog队列中的数据将通过\_\_netif\_receive\_skb（实际处理发生在\_\_netif\_receive\_skb\_core）函数出队并传递给网络协议栈。\_\_netif\_receive\_skb\_core首先得到接收数据包的协议类型字段，然后遍历为该协议类型注册的传递函数列表进行数据传送。[部分源码](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L3548-L3554)    

	type = skb->protocol;
    list_for_each_entry_rcu(ptype, &ptype_all, list) {
		if (!ptype->dev || ptype->dev == skb->dev) {
			if (pt_prev)
				ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = ptype;
		}
	}
#### 协议栈处理
接下来，数据包将依次经过Linux内核中的TCP/IP协议栈，在处理完成之后添加到socket缓冲区（或者被转发、丢弃等）、等待应用层程序的读取。该过程将会在后续的文章中详细分析。   

### 附
[Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)     


	