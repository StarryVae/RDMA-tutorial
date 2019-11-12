# Performance Isolation Anomalies in RDMA

这篇发表在SIGCOMM 2017的workshop KBNets上的文章算是我第一篇读懂的关于RDMA的文章，也是领我入门的一篇文章。因为这篇文章侧重于实验的对比，而且提供了实验对比的源码，我可以在自己的平台上去跑文章中的实验，进一步分析源码，深刻理解在一般的科研过程中，如何结合已有的RDMA编程的知识去设计RDMA实验。

当然这篇文章的主要贡献不在于实验，而是通过实验验证的RDMA存在的一个问题：不同应用之间的性能隔离问题。简而言之就是当多个应用共享RDMA网络时，彼此之间是否会有影响？有怎样的影响？原因可能是什么？那么为了得到答案，文章将应用主要分为吞吐量敏感和延时敏感两类，其中，数据包大小大于1MB的流称为吞吐量敏感的大流，而数据包大小小于1KB的流称为延时敏感的小流，通过不同数据包大小的流来模拟不同的应用发包。实验主要分为三种：大流与大流竞争，大流与小流竞争，以及大流与小流竞争。

### 大流 vs 大流

如下图所示，分别测试了在busy poll和epoll方式下不同大流之间的竞争情况，可以发现以下问题：

* 在epoll方式下，数据包大小更大的流能获得更多的带宽。
* 在busy poll方式下，性能隔离性能得到保障。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Performance Isolation Anomalies in RDMA 1.jpg" width = 70%>
</div>

### 大流 vs 小流

大流使用epoll，小流使用busy poll，从下图可以看到小流的延时和吞吐量性能会急剧下降，大流的相关性能则不会受影响。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Performance Isolation Anomalies in RDMA 2.jpg" width = 70%>
</div>

### 小流 vs 小流

小流与小流的竞争即使是在数据包大小一样的情况下都是不可预测的。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Performance Isolation Anomalies in RDMA 3.jpg" width = 70%>
</div>

### 结论

* 大流与大流的竞争与poll的策略有关，也就是与应用将RDMA请求发送给网卡的速率有关
* 大流与小流的竞争由于小流的wqe可能需要排队等大流的wqe，因此延时和吞吐量性能下降。
* 小流与小流的竞争由于各自的wqe都能很快的被处理，因此充满着不可预测性。


这篇文章当时对我最大的作用就是在实验方面，让我了解了RDMA对比实验是如何设计的，从而进一步加深了对RDMA编程的理解。当然现在来看，会有一些其他的感悟和想法。首先当时在学习RDMA的时候也了解过infiniband链路层的一些知识，知道有virtual lane的概念，其实就是具有优先级的一些虚拟链路，理论上来说是可以根据需要自己配置来实现流量之间的隔离，但是当时尝试后失败，而且这篇文章中也提到这种方式不可行。然后本身自己对于RDMA网卡内部的QP、wqe的调度策略不是很了解，那么是否可以在应用的层面去实现一些公平的调度策略？至少先能提供这样一个调度的框架，具体的策略或者说是调度模型可以再往框架中添加？现在来看是可以的，并且结合上RDMA的可扩展性问题带来的QP资源共享，这样的公平性调度将更加有意义。