# On the Impact of Cluster Configuration on RoCE Application Design

我只能说这是一篇很有意思的文章，因为他跟我第一个点的想法几乎一模一样，而且我们同时投了APNet 2019，结果这篇文章中了，我只中了poster，于是仔细“拜读”了这篇文章。

文章依然是针对目前数据中心应用和RDMA结合的性能问题展开讨论，主要讨论以下几个之前文章没研究过的问题：

* 应用之间对CPU资源的竞争会给RDMA性能带来什么样的影响？
* 在考虑CPU资源竞争的情况下如何为应用选择RDMA verbs？
* 应用之间对RNIC资源的竞争会给RDMA性能带来什么样的影响？

接着文章通过各种对比实验来分析上面的问题，得到如下的一些现象和结论：

* CPU竞争会改变最佳verb的选择
* 选择最佳的verb是很困难的，与很多因素有关：payload大小、CPU使用情况、RNIC使用情况、内存访问的次数等
* RDMA应用不能公平的进行资源共享
* busy polling只在没有CPU竞争的情况下能带来更好的性能，因此poll的机制应该要随着CPU的使用情况动态切换

最后，文章觉得在设计数据中心RDMA应用的时候需要考虑太多的因素，因此需要一个higher-level的库来屏蔽掉这些复杂的因素，为应用提供高性能的接口，如下图所示，我和作者用英文尬聊过，据说是后面要实现这个库发NSDI。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Cluster Configuration.png" width = 70%>
</div>

整个看下来，其实主要内容和我都是差不多的，包括提到的payload大小、CPU竞争、busy/event-triggered polling的切换等在我文章里都有涉及，可能主要的差距在于他的实验图更完善？英文写的更好？突出了CPU竞争的新发现？而我的主题是实时调整的RDMA参数，没有去突出其中CPU资源变化带来的参数调整？但总的来说，当时开完会觉得自己做的东西还不错，至少类似的内容得到了肯定。