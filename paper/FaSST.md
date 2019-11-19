# FaSST: Fast, Scalable and Simple Distributed Transactions with Two-sided (RDMA) Datagram RPCs

FaSST也是Anuj Kalia的文章，发表在OSDI 2016上。细看这篇文章，会发现和HERD有很多相似之处，同时增加了一些RDMA非常细节的优化点，比如Doorbell batching、Cheap RECV posting等。

这篇文章的背景是针对分布式内存事务应用的，当前的工作利用one-sided的RDMA元语进行分布式事务的数据传输，这种方式主要存在以下两个问题：一个是one-sided的元语缺乏灵活性，在软件开发方面非常复杂；另一个是可扩展性的问题（类似HERD）。为了解决这些问题，文章提出基于two-sided的元语设计RPC用于分布式事务的数据传输，和HERD不同的是，本文是针对all to all的场景，也就是服务器既是client也是server，因此完全是使用基于UD的two-sided元语进行数据的传输，最终带来了题目中的Fast, Scalable and Simple的优势。

## Fast

Fast指的是HERD中提到的READ需要多个round trip，而two-sided的元语只需要一个round trip，而且当规模扩大时，因为cache miss的问题，one-sided元语性能会急剧下降，而基于UD的two-sided元语却不会受影响。除此之外，Doorbell batching也是UD的一大优势，Doorbell指的是应用在post wqe的时候会通过PCIe writing
to a per-QP Doorbell register on the NIC，而这个write操作是比较消耗CPU资源的。因此在使用one-sided元语时，the process must ring multiple Doorbells—as many as the number of destinations appearing in the batch，而With a datagram QP, the process only needs to ring the Doorbell
once per batch, regardless of the individual message destinations within the batch。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/FaSST1.jpg" width = 70%>
</div>

## Scalable

Scalable顾名思义，基于UD的two-sided元语可扩展性很好，因为UD支持一对多的数据传输，不需要创建多个QP。而基于RC/UC的one-sided元语只支持一对一的QP连接，因此需要大量的QP，RNIC就会溢出，性能下降。

## Simple

Simple指的就是two-sided元语和传统网络的socket编程很类似，有receive和send的过程，实现起来相比WRITE/READ要简单的多。

自己平时学习RDMA，写RDMA程序，其实都是根据现有的接口实现自己的逻辑，从来没有去思考过这些接口底层是怎么操作的，这些底层的属性是不是也会影响到上层应用的性能？当然后来经历过找工作后，也算是形成了这样的思维。这篇文章的主体思想其实在看过HERD之后没有感到很特别，但是它的细节却是让我又一次折服Anuj Kalia的专业。