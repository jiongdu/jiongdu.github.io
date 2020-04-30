---
title: Logging、Metrics和Tracing
date: 2018-12-15 19:26:20
tags: DevOps
---
近来研究了一系列面向DevOps的诊断与分析系统，并在部门结合业务场景逐步落地。本文记录了该过程中的一些思考和总结。

<!--more-->
### 三类系统的诞生
随着DevOps、微服务、容器等理念的逐步落地和发展，传统软件架构正在发生着较大的变化，其主要体现在：
- 应用架构开始从单体系统逐步转变为微服务，其中的业务逻辑随之会变成微服务之间的调用与请求。
- 从资源的角度来，服务器这个物理单元正在淡化，取而代之的是看不见摸不着的虚拟资源模式。

![](http://km.oa.com/files/photos/pictures/201810/1538896485_13_w694_h347.png)

以上两个变化使得分布式系统的运维与监控变得越来越复杂。为了应对这一趋势，诞生了一系列面向DevOps的诊断与监控系统，包括三大类：
- 集中式日志系统（**Logging**）
- 集中式度量系统（**Metrics**）
- 分布式追踪系统（**Tracing**）

### 一些疑问
对Logging、Metrics和Tracing三种系统有了初步认识和了解之后，我对它们的关系产生了疑问：
- Logging和Metrics的区别是什么？
- 什么时候需要Metrics？
- Tracing和Logging之间有什么关联？
- 三种系统各自解决的主要问题是什么？有无交集？

### 一些思考
![](http://km.oa.com/files/photos/pictures/201810/1538897894_21_w392_h400.jpg)
- Logging是以**事件**为元数据，记录某个时刻（离散时间）发生的事件，比如将应用的调试或错误信息发送到ElasticSearch，跟踪时间信息通过消息队列处理发送到数据仓库等。它的特点是大多数情况下记录的数据很分散，并且相互独立，也许是错误信息，也许仅仅是记录当前的事件状态等，这是Logging的关注属性。
- Metrics有其特有的属性---**可聚合性**。比如我们想知道服务请求的QPS，用户的登录次数等指标，这时我们需要将一部分事件进行聚合或计数，这就是Metrics。它是一段时间内某个度量（计数器或直方图）的元数据。
- 在记录Tracing数据的时候，或多或少会有Logging的数据，比如在Tracing中的链路数据指标（rpc调用，请求处理时间是多少等），同样会被记录在Logging中，但Tracing同样也有自己独特的属性---**请求范围**，即可以绑定到系统单个“事务”对象的生命周期的任何元数据。例如一次HTTP请求的tracing，我们只需要关注请求到达时的节点状态，接收方和处理耗时等，不用关心具体的系统日志、错误信息等Logging事件。

因此，根据上述的描述，我们可以标记上图中三者的重叠部分。
![](http://km.oa.com/files/photos/pictures/201810/1538899025_24_w676_h400.png)
当然，大量被诊断的应用具有分布式的能力，逻辑处理在单次请求的范围内完成。因此，讨论追踪的上下文是有意义的。但是，我们注意到，并不是所有的诊断系统都绑定在请求的生命周期上的。他们可能是逻辑组件诊断信息、处理过程的生命周期明细。所以，不是所有的metrics和logging都可以塞进tracing系统的概念中，至少在不经过数据处理是不行的。
此外，上图中我们还能看到，在三个功能域中，metric倾向于更节省资源，因为它会“天然”压缩数据。相反，日志倾向于无限增加，会频繁的超出预期的容量，所以，图中纵向绘制了容量的需求趋势，metrics低到logging高。

### 业界代表性系统
- **ELK**
	* 简介：专注于**Logging**领域，由ElasticSearch、Logstash和Kibana三款软件组成的一整套解决方案，提供了日志的收集、传输、存储和分析等功能。该架构由Logstash分布于各个节点上搜集相关日志、数据，并经过分析、过滤后发送给远端服务器上的ElasticSearch进行存储，用户通过直观地配置Kibana对日志进行可视化查询、生成报表等。该架构现已在部门内落地。
	* 官网：https://www.elastic.co/cn/elk-stack
- **Prometheus**
	* 简介：主要聚焦**Metrics**领域（正在集成更多的tracing功能），既适用于面向服务器等硬件指标的监控，也适用于高动态的面向服务架构的监控。Prometheus的基本原理是通过HTTP协议周期性抓取被监控组件的状态，任意组件只要提供对应的HTTP接口就可以介入监控，不需要任何SDK或者其他的集成过程。
	* 官网：https://prometheus.io/
- **Zipkin**
	* 简介：专注于**Tracing**领域，根据Google Dapper论文设计的全链路监控系统，由Twitter公司开发。每个服务向Zipkin报告及时数据，Zipkin会根据调用关系通过UI生成依赖关系图，显示有多少跟踪请求通过每个服务，该系统让开发者可通过Web轻松的收集和分析数据，例如用户每次请求服务的处理时间，可方便的检测系统中的瓶颈。
	* 官网：https://zipkin.io/
	
### 参考
https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html
https://yq.aliyun.com/articles/394183
https://techbeacon.com/monitoring-demystified-top-logging-tracing-metrics-resources

