---
title: NVMe驱动之I/O请求
date: 2017-04-27 20:31:41
tags: NVMe over RDMA
---
NVMe SSD具有极高的I/O读写性能，不存在传统磁盘所具有的访问寻道、抖动问题。为了发挥NVMe SSD的性能，无论在软件还是在硬件上都需要采用多队列技术，通过多队列方式充分发挥NVMe SSD的性能。
<!--more--> 
Linux的NVMe驱动采用一个Core独占一个Queue（包含一个Completion Queue和至少一个Submission Queue）的方式来充分发挥NVMe SSD的性能，这样可以避免一个队列被多个Core竞争访问（引起的锁竞争等问题），各CPU使用自己的Queue，互不影响。 
     
![](http://i.imgur.com/sIdEXDh.gif)   

NVMe响应I/O的过程已经在[NVMe技术浅析](http://blog.dujiong.net/2017/02/17/NVMe/)一文中详细描述，这里将结合部分源码做进一步说明。    
### 创建NVMe Queue
	
	/*
     * pci.c
     */
	static int nvme_setup_io_queues(struct nvme_dev* dev){
		struct nvme_queue* adminq = dev->queues[0];
		struct pci_dev* pdev = to_pci_dev(dev->dev);
		int result, nr_io_queues, size;

		nr_io_queues = num_online_cpus();
		result = nvme_set_queue_count(&dev->ctrl, &nr_io_queues);
		...
		
		result = queue_request_irq(adminq);
		if(result){
			admin->cq_vector = -1;
			return result;
		}
		return nvme_create_io_queues(dev);
	}

驱动程序在创建队列之前首先通过内核函数num\_online\_cpus获取当前系统中运行的CPU数量m，然后通过nvme\_set\_queue\_count得到SSD的队列数量限制n，选择二者较小值作为所创建的队列数。    
此外，关于Queue还有一个重要的概念，那就是队列深度，简单来说就是这个Queue能够放多少个成员（比如NVMe Command）。在NVMe中，这个队列深度是由NVMe SSD决定的，存储在NVMe设备的BAR空间里。另外，NVMe驱动没有区分Namespace，也就是一个设备上的多个Namespace共享这些队列资源。     
因此，它们之间的关系是：队列用来存放NVMe Command，NVMe Command是Host与SSD Controller交流的基本单元，应用的I/O请求也要转化成NVMe Command。
### 提交I/O请求  
Block层下发的IO请求以BIO表示，我们需要通过DMA发送这些数据，Command使用dma\_alloc\_coherent分配DMA地址，但是BIO是存放在普通的内核线程空间的（线程的虚拟空间不能直接作为DMA地址），使用dma\_alloc\_coherent分配再将BIO中的数据拷贝显然不是有效的方法。这里linux提供了另一个函数dma_map_single，这个函数能够将虚拟空间地址（BIO数据存放地址）转换成DMA可用地址，并且多个IO请求的DMA地址可以通过scatterlist来表示。有了DMA地址就可以把BIO封装成NVMe Command发送出去。从BIO到NVMe Command有可能会经过拆分，放入等待队列等。我们知道，驱动会给每个CPU分配一个Queue，那么I/O请求到来时，该由哪个Queue来存放这个Command呢？你可能已经想到，应该由当前线程运行的CPU所属的Queue，这样才能保证Queue不被其他Core抢占，驱动使用get\_cpu()获得当前处理I/O请求的CPU号来索引对应的Queue。 

	/*
     * pci.c
     */
	static void __nvme_submit_cmd(struct nvme_queue* nvmeq, struct nvme_command* cmd){
		u16 tail = nvmeq->sq_tail;
		if(nvmeq->sq_cmds_io){
			memcpy_toio(&nvmeq->sq_cmds_io[tail], cmd, sizeof(*cmd));
		}else{
			memcpy(&nvmeq->sq_cmds[tail], cmd, sizeof(*cmd));
		}

		if(++tail == nvmeq->q_depth)
			tail = 0;
		writel(tail, nvmeq->q_db);
		nvmeq->sq_tail = tail;
	}
   
BIO封装成的Command会顺序存入Submission Queue中。对于Submission Queue来说，使用Tail表示最后操作的Command Index（nvmeq->sq_tail）。每存入一个Command，Host就会更新Queue对应的Doorbell寄存器中的Tail值。Doorbell定义在BAR空间，通过QID可以索引到。NVMe没有规定Command存入队列的执行顺序，Controller可以一次取出多个Command进行批量处理，所以一个队列中的Command执行顺序是不固定的（可能导致先提交的请求后处理）。
     
### 获得处理结果     
SSD Controller根据Doorbell的值，获取NVMe Command和对应数据，待处理完成后将结果存入Completion Queue中。Controller通过中断的方式通知Host，驱动为每一个Queue分配一个MSI/MSI-X中断。   
驱动使用request\_irq将中断注册到kernel，并且绑定中断处理函数nvme\_irq。此中断处理函数调用\_\_nvme_process\_cq，先将Completion Command从Completion Queue中取出，然后把队列的head增加1，并调用上层的callback函数，完成IO处理。由于NVMe Command可以批量处理，这里使用while循环取出所有新的Completion Command。     

	/*
	 * pci.c
     */
	static void __nvme_process_cq(struct nvme_queue *nvmeq, unsigned int *tag){
		u16 head, phase;

		head = nvmeq->cq_head;
		phase = nvmeq->cq_phase;

		while(nvme_cqe_valid(nvmeq, head, phase)){
			struct nvme_completion cqe = nvmeq->cqes[head];
			struct request *req;
			
			if(++head == nvmeq->q_depth){
				head = 0;
				phase = !phase;
			}
			...
			if (unlikely(nvmeq->qid == 0 &&
				cqe.command_id >= NVME_AQ_BLKMQ_DEPTH)) {
					nvme_complete_async_event(&nvmeq->dev->ctrl,
											cqe.status, &cqe.result);
					continue;
			}
			req = blk_mq_tag_to_rq(*nvmeq->tags, cqe.command_id);
			nvme_req(req)->result = cqe.result;
			blk_mq_complete_request(req, le16_to_cpu(cqe.status) >> 1);	
		}
		if (head == nvmeq->cq_head && phase == nvmeq->cq_phase)
			return;

		if (likely(nvmeq->cq_vector >= 0))
			writel(head, nvmeq->q_db + nvmeq->dev->db_stride);
		nvmeq->cq_head = head;
		nvmeq->cq_phase = phase;

		nvmeq->cqe_seen = 1;
	}

处理完Command后，往Completion Queue的Doorbell写入Head值，通知NVMe Controller操作完成。中断处理结束。

### Head/Tail机制
我们知道，SQ和CQ都是队列，队列的头尾很重要，头决定谁会最先被服务，尾决定新到来命令的位置，所以，我们需要记录SQ和CQ的头尾位置。  
    
![](http://i.imgur.com/bsa3KFs.png)

对一个SQ来说，它的生产者是Host，因为它往SQ的Tail位置写入命令；消费者是SSD，它从SQ的Head取出指令执行。CQ则刚刚相反，生产者是SSD，消费者是Host。     

### 总结
本文简单介绍了NVMe I/O请求的处理过程，其核心是多队列，Submission Queue/Completion Queue、Doorbell寄存器和Head/Tail机制。