# RDMA Tutorial

## 一、什么是RDMA

### 1.1 RDMA总览

随着以太网的提速，TCP/IP 协议基于软件协议栈传输的这种方式（图1）已经无法适应高速数据传输的需求，成为网络性能进一步增长的瓶颈。应用数据在用户态和内核态之间的拷贝带来ms级的延时，协议栈对数据包的解析、寻址、校验等操作需要消耗大量CPU资源。

普通网卡的工作过程如下：先把收到的数据包缓存到系统上，数据包经过处理后，相应数据被分配到一个TCP连接。然后，接收系统再把主动提供的TCP数据同相应的应用程序联系起来，并将数据从系统缓冲区拷贝到目标存储地址。这样，制约网络速率的因素就出现了：应用通信强度不断增加和主机CPU 在内核与应用存储器间处理数据的任务繁重，使系统要不断追加主机CPU 资源，配置高效的软件并增强系统负荷管理。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/软件协议栈.png">
</div>

新型的网络技术替代了传统的TCP/IP软件协议栈的设计，核心技术就是RDMA，全称是远程直接内存访问，实现了kernel bypass技术，数据不需要经过软件协议栈，并且不需要CPU参与寻址、解析等操作，从而提供低延时、高带宽、低CPU使用的性能优势：

* Remote：远程服务器之间的数据交换
* Direct：内核旁路技术、协议栈下发到网卡
* Memory：用户态应用虚拟内存
* Access：SEND/RECV，READ，WRITE等操作

如图2所示，从RDMA的宏观传输图中，我们可以看到，数据直接从用户态发送给网卡，再由网卡中的协议栈进行转发到达目标端用户态内存，整个过程完全旁路了内核，不需要用户态到内核态的数据拷贝，降低了延时，同时不经过软件协议栈，也就不需要CPU参与寻址等操作，较少了CPU的使用。在这个过程中，RNIC是最重要的一个设备，那么接下来先介绍RNIC。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA宏观传输图.png">
</div>

### 1.2 RNIC

RNIC即RDMA网卡，是RDMA实现kernel bypass的专有硬件，那么RNIC主要又由哪些部分所组成呢？如图3所示，RNIC主要包括QP以及CQ：

QP：Queue Pair，一组工作队列，由发送队列SQ和接收队列RQ组成，主要用于接收应用发起的发送请求和接收请求；

CQ：Completion Queue，完成队列，当发送请求或接收请求完成时都会产生一个完成的通知。

各个队列的具体工作流程会在下面介绍RDMA编程时详细介绍。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RNIC组成.png">
</div>

## 二、RDMA编程