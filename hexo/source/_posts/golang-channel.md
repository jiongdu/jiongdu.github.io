---
title: golang中channel的属性分析
date: 2018-09-15 19:26:20
tags: golang
---
我将golang中的channel理解为信号，即channel允许一个goroutine向另一个goroutine发出某些特定事件的信号。

<!--more-->
**信号是理解golang中使用channel做一切工作的核心。**
本文将从channel的三个关键属性出发来理解信号机制是如何工作的。这三个属性分别是：
**1、可靠性传递
2、状态
3、是否含有数据**
下面将结合图示和代码分别对其进行分析。
## 可靠性传递
属性“可靠性传递”是基于信号传递中的一个常见问题：“我如何得知一个goroutine发出的特定信号已被正确接收？”
```go
1 go func() {
2	p := <-ch		
3 } ()
4
5 ch <- "paper"
```
以上面的代码为例，我们可以根据发送信号的goroutine（第5行）是否需要（**立即**）得到接收信号的goroutine（第2行）的**确认**（Guaranteed）来将channel划分为Unbuffered（无缓冲的）和Buffered（带缓冲的）两种类型，它们针对“可靠性传递”这一属性提供了不同表现。如下图所示。
![](http://km.oa.com/files/photos/pictures/201806/1529824533_88_w681_h152.jpg)
“带确认的可靠性传递”在很多场景下是非常重要的，这里先做简单说明，后文还将更详细地介绍。
## 状态
channel的行为受其当前所处状态的影响，channel可能处于的状态包括open，closed和nil。
下面的代码实例中展示了三种可能的状态：
```go
var ch chan string				//nil state
ch = nil							//nil state
ch := make(chan string)  	 		//open state
close(ch)								//closed state

```
channel所处的状态决定了对其进行send和receive操作时的表现（如前文所述，我们将channel理解为信号的行为，因此将操作称为send和receive，而不说read和write）。如下图所示。
![](http://km.oa.com/files/photos/pictures/201806/1529825902_47_w908_h227.jpg)
（1）当一个channel处于nil状态时，任何试图对channel进行的send/receive操作都将被阻塞；
（2）当一个channel处于open状态时，信号可以在channel中被发送和接收；
（3）当一个channel处于closed状态时，不能再发送信号，**但仍可能接收信号**。
## 是否含有数据
最后一个需要考虑的属性是信号中是否含有数据。
当信号中含有数据时（`ch <- "paper"`），通常发生在：
（1）一个goroutine需要开启一项新任务；
（2）一个goroutine需要反馈执行结果。
而当信号中不含有数据时（`close(ch)`），通常发生在：
（1）一个goroutine需要被停止；
（2）一个goroutine需要反馈其已经完成或关闭了处理。
### 含有数据的信号
当使用含有数据的信号时，我们可以根据需要得到的确认类型来选择channel的配置，可选的三个配置分别是Unbuffered、Buffered>1和Buffered=1，其特点如下图所示。
![](http://km.oa.com/files/photos/pictures/201806/1529827175_43_w908_h152.jpg)
（1）Unbuffered channel：提供了可靠性确认，确认发送的信号已被接收，因为信号的接收发生在信号发送完成之前；
（2）Buffered>1：不能保证发送的信号已经被接收，因为信号的发送在信号接收完成之前结束；
（3）Buffered=1：提供了延迟确认，可以保证先前发送的信号已经被接收，因为第一个信号的接收发生在第二个信号发送完成之前。
### 不含有数据的信号
不含有数据的信号主要表现为取消操作，它允许一个goroutine向另外的goroutine发出信号来停止或继续它们正在做的事情。该操作可以通过Buffered和Unbuffered两种channel实现，但相对来说Buffered channel更常见一些。
不含有数据信号操作的一个常见例子是内置函数close。如前文所述，我们仍然可以在close后的channel上接收信号，并且不会阻塞，总是会返回。
## 实际应用场景
基于以上对channel三个关键属性的总结，接下来通过实际代码中的一些场景来对channel做进一步理解。在下面的几个例子中，我们将goroutine抽象成人，将channel抽象成人与人之间的信号传递。
### 带确认+含有数据+Unbuffered Channel
我们通过两个实际场景来说明该类型Channel的工作机制。
#### 场景1：等待任务
想象一位经理面试一位新雇员的场景。该场景下，经理希望雇员做一些试题以检测其工作能力，雇员需要拿到经理递去的正式答题纸后方可开始作答。

```go
01 func waitForTask() {
02	 ch := make(chan string)
03	 
04	 go func() {
05		p := <-ch
06		//新雇员开始答题
07		//新雇员答题后离开
08	 }()
09	 time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
10	 ch <- "paper"
11 }
```
在第02行，程序创建了一个带string数据的Unbuffered channel，然后在第04行，新雇员在开始答题之前被告知在05行等待经理的正式答题纸，这将使得雇员在等待时阻塞。而一旦雇员收到公司的正式答题纸，他就可以开始作答，并在完成之后离开。而经理与雇员在并发工作，当准备好正式答题纸之后，经理向雇员发出信号，即第10行的数据“paper”，由于使用的是Unbuffered channel，可以保证一旦发送操作完成后，新雇员就立即收到了数据“paper”。
#### 场景2：等待结果
在接下来这个场景中，情况发生了变化。这一次，经理希望新雇员在被雇佣时立即执行答题（所有必备材料已准备好），而经历需要等待雇员的答题结果，待其完成之后，再进行自己的工作。
```go
01 func waitForResult() {
02	ch := make(chan string)
03	
04	go func() {
05		time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)  //雇员答题
06		
07		ch <- "paper"			//雇员完成答题，并离开
08	}()
09	ch := <-ch
10 }
```
在第02行，程序创建了一个带string数据的Unbuffered channel，然后在第04行，新雇员开始面试并立即投入答题。而经理在09行等待着其答题结果。一旦答题在05行结束，新雇员通过07行的数据发送告知经理，由于这是Unbuffered channel，接收发生在发送完成之前，因此新雇员保证经理已经接收到答卷。在这种场景下，经理不知道新雇员需要多长时间才能完成答题。
### 无确认+含有数据+Buffered>1 Channel
有时我们不需要知道被发送的信号是否被（立即）接收，比如下面的两种场景。
#### 扇出
扇出模式允许经理让多名员工同时工作并完成报告，经理需要确保自己有足够的空间来容纳这么多的报告。
想象一位经理面试一个员工团队的场景。经理希望团队中的每位员工都执行一个单独的任务，当员工完成任务时，他们需要为经理提供一份报告，并放置在经理办公桌上的盒子里。
```go
01 func fanOut() {
02	emps := 20
03	ch := make(chan string, emps)
04	
05	for e := 0; e < emps; e++ {
06		go func() {
07			time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
08			ch <- "paper"
09		}() 
10	}
11
12	for emps > 0 {
13		p := <-ch
14		fmt.Println(p)
15		emps--
16	}
17 }
```
在第03行，程序创建了一个带string数据的Buffered channel，这一次，该channel的缓冲区大小设定为20。在第05行至10行之间，20名新雇员立即开始工作，经理不知道每位员工完成任务需要多少时间（第07行）。然后在第08行，员工完成任务后发送报告，但该发送过程并不阻塞等待接收。因为盒子里每位员工都有空间，channel上的发送只与其他在同一时间发送报告的员工竞争。第12行至16行之间的代码表示经理等待所有20名员工完成他们的工作并发送报告。一旦收到报告，将在第14行打印，并且本地计数器变量递减。
#### Drop
Drop模式允许经理在雇员有空闲时继续布置任务，即雇员有一个能力值，在能力值范围之内可以继续接收任务，超出能力范围则只能Drop。
想象一位经理安排员工完成任务的场景。经理与员工之间没有直接接触，而是通过一个存放任务的箱子沟通，如果经理能将新工作（任务书）放进箱子里，表明员工还能继续承担任务；如果不能执行发送，表明箱子已满，新的工作只能被丢弃。
```go
01 func selectDrop() {
02	const cap = 5
03	ch := make(chan string, cap)
04	
05	go func() {
06		for p := range ch {
07			fmt.Println("employee : received : ", p)
08		}
09	}()
10	
11	const work = 20
12	for w := 20; w < work; w++ {
13		select {
14			case ch <- "paper":
15				fmt.Println("manager : send ack")
16			default:
17				fmt.Println("manager : drop")
18		}
19	}
20	close(ch)
21 }
```
在第03行，程序创建了一个带string数据的Buffered channel，这一次，该channel的缓冲区大小设定为5。在第05行至09行之间，经理雇佣一名新员工来处理工作，`for range`用于信道接收的范围，每次收到'paper'，就在07行处理。在第11至20行之间，经理尝试向员工发送20张'paper'，这里使用SELECT语句实现，如果缓冲区没有更多的空间（`default`），发送将被阻塞，在第17行放弃发送。最后，在第21行，内置函数close被调用关闭channel，这将在没有数据的情况下向完成工作的员工发送信号。
### 延迟确认+含有数据+Buffered=1 Channel
有些时候在发送信号之前需要知道对端是否接收到发送的前一个信号，比如下面的“等待接收任务”场景。
#### 等待接收任务
该场景下，经理雇佣了一位新员工，并一个接一个地为他安排任务。但是，雇员必须在完成一项任务之后才能开始新的任务。
此时，如果经理和雇员都按照预定的节奏进行，那么他们不需要额外的互相等待。每次经理发送信号的时候，缓冲区都是空的，每次雇员要开始新的任务时，缓冲区都是满的。
```go
01 func waitForTasks() {
02	ch := make(chan string, 1)
03	
04	go func() {
05		for p := range ch {
06			fmt.Println("employee : working :", p)
07		}
08	}()
09	
10	const work = 10
11	for w := 0; w < work; w ++ {
12		ch <- "paper"
13	}
14	
15	close(ch)
16 }
```
在第02行，程序创建了一个带string数据的大小为1的Buffered channel，在第04行至08行之间，一个新员工被雇佣来处理工作，`for range`用于channel接收信号，每次收到信号，就在第06行处理。
在第10行至13行之间，经理开始向员工安排任务，如果员工可以以尽可能快的速度完成，那么经理与员工之间的等待时间就会减少。但是每一次发送成功，经理都能保证其提交的最后一份工作正在被处理。
最后，在第15行，内置函数被调用关闭channel。这将向员工发出信号已经完成了工作。

## 总结
可靠性传递、信道状态和是否含有数据这三大属性对于理解和使用channel非常重要，本文通过一些实例代码和示意图对这些属性进行了分析，希望能帮助我们更好地掌握channel的工作机制以编写良好的并发程序。

### 附
[Go Channel详解](http://colobu.com/2016/04/14/Golang-Channels/)
[The Behavior of Channels](https://www.ardanlabs.com/blog/2017/10/the-behavior-of-channels.html)
[深入理解Go Channel](http://legendtkl.com/2017/07/30/understanding-golang-channel/)

