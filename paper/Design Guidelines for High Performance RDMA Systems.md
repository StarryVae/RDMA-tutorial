# Design Guidelines for High Performance RDMA Systems

Anuj Kalia对RDMA的研究越来越底层，这篇文章的Design Guidelines不是简单的连接类型、元语选择的guideline，而是从PCIe和网卡结构方面考虑影响RDMA性能的因素，最后反映到和具体应用结合时的一些设计细节，为本来的RDMA应用进一步带来性能上的提升。那我们具体来看文章中提到的RDMA design guidelines：

## Reduce CPU-initiated MMIOs

CPU通过MMIO发消息给网卡来发起网络操作，这个消息可以包括 1）新的WQE；2）对新的WQE的引用。其中第一种类型的消息通过MMIO来传输（WQE-by-MMIO），第二种消息通过一次或者多次的DMA来读WQE（Doorbell）。WQE-by-MMIOs方法是为低延迟做优化，通常是默认的方法。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/design1.jpg" width = 70%>
</div>

两种优化可以通过减少MMIOs来改善性能：

* Doorbell batching：应用一次发出多个WQE到一个QP，它可以使用一次Doorbell MMIO来做一次batch
* WQE shrinking：减少被一个WQE使用的缓存行的数量可以大大地改善吞吐量。

## Reduce NIC-initiated DMAs

已知的用来减少NIC发起的DMAs包括unsignaled verbs来减少完成的DMA写（CQE）和负载内联来避免负载DMA读（inline data）。这篇文章还发现NIC必须对每一个完成的RECV DMA一个CQE。和其他种类那种只提醒完成并且是不重要的的verbs不一样，RECV的CQEs包含重要的元数据，比如收到数据的大小。NIC通常会为负载和完成产生两个分离的DMA，分别把他们写到应用程序享有的内存和驱动享有的内存里。我们假设对应一个RECV的SEND携带X字节的负载，那么还存在以下两种优化：

* Inline RECV：如果X很小，NIC把负载封装进CQE里，这个CQE后来会被驱动复制到应用程序指定的地址空间里去。
* Header-only RECV：如果X=0，就是没有负载的情况下，在接收方就不会产生负载的DMA。报文的头部里的一些信息被包含在DMA发出的CQE中，可以用来实现应用协议。那么就只会产生一次DMA而不是两次DMA。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/design2.png" width = 70%>
</div>

## Engage multiple NIC PUs

把一个cpu核绑定到一个PU上会把CPU的吞吐量限制在PU的吞吐量上，比如当某个应用对每个数据包的处理非常少，在这种情况下，每个核使用多个QP会增加CPU效率，这叫做多队列优化。

## Avoid contention among NIC PUs

这个guideline我不是很理解，大概是指NIC PUs之间的锁竞争会带来性能的损耗？

## Avoid NIC cache misses

这个就是之前HERD、FaSST中都提到的cache miss的问题，主要解决方法包括：使用大页、使用更少的QP。

以前看这篇文章的时候很蠢，只能get到Reduce NIC-initiated DMAs中提到的inline和unsignal，并且把这两个参数放入到了自己第一个研究点的框架当中，而其中涉及的batching、multiple NIC PUs的优化因为看不懂也没在意。现在看来，这些机制对RDMA性能有更大的提升！Anuj Kalia这些研究工作对真实系统有性能上的提升才是真正有意义的研究工作！