# Deconstructing RDMA-enabled Distributed Transactions: Hybrid is Better!

这篇文章因为和我第一个点的想法有点类似，在找工作期间稍微了解了一下，最近终于有时间仔细研究研究。当时看到文章的标题Hybrid is Better就感觉很有意思，难道是指RDMA参数的混合能给应用带来更好的性能？果然是这样。

这篇文章是针对分布式事务应用利用RDMA进行加速，和之前文章不同的是，这篇文章不在纠结于是one-sided还是 two-sided元语更适用于分布式事务，而是“phase-by-phase”逐阶段的分析分布式事务在execution、validation、commit、logging阶段分别适合使用哪些元语。那其实就完美的佐证了我第一个研究点的主要内容：固定的RDMA参数并不总能给应用带来最佳的性能，RDMA参数需要在应用运行时根据应用的特性和服务器资源状态进行调整。当然，针对具体应用的分阶段参数选择可能会比我这种通用的框架效果更好，因为它们会根据应用的执行逻辑进一步选择RDMA参数，这在通用的框架里是很难实现的。

这篇文章除了提出了很有意思的混合参数的想法，另一个对于我来说新的发现就是文章中提到的“Passive ACK”。对于one-sided元语来说，指的就是unsignal，这个不用多说；主要是针对two-sided元语来说，如下图所示，在对称传输的场景下，请求的完成可以通过对方的回复而得知，从而减少一半的数据包，但这个感觉是应用层次上的优化，和RDMA没啥关系？在传统网络里也可以这么做？

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/hybrid.jpg" width = 70%>
</div>

这篇文章出自上交的IPADS实验室的博士，自叹不如，强大的系统分析和设计能力，越来越觉得自己很弱，只能以他们实验室早在很多年前就研究RDMA且人多力量大为借口安慰自己。。