# Improving the Performance of Distributed TensorFlow with RDMA

这篇文章相比于HERD来说就显得简单粗暴一点，直接用RDMA的WRITE和READ替换了tensorflow的gRPC通信框架，并取得了极大的性能提升。当然，tensorflow这种深度学习框架本身就很复杂，能够将它与RDMA相结合在我看来本身就是挺有挑战性的一件事，更不用说如果像HERD一样去进一步分析tensorflow的特性，选择最佳的RDMA参数。

这篇文章的背景就是针对现在GPU设备的出现，集群计算能力增强的情况下，在进行分布式深度学习训练的时候，传统网络成为了较大的瓶颈。并且在TCP/IP环境下利用tensorflow进行了Alexnet模型的分布式训练实验，发现分布式训练的时间反而比单机训练时间长，这也就证明了TCP/IP网络带来的瓶颈。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/tensorflow2.jpg" width = 70%>
</div>

那么为了解决这个问题，文章的目标就是利用RDMA替换tensorflow基于TCP/IP的gRPC通信框架。首先是分析了tensorflow的send/receive流程，如下图所示，确定需要替换的模块。接着就分别利用RDMA的WRITE以及READ元语进行替换。最后测试性能的提升。整个文字的介绍过程很简单，但工程量应该是挺大的。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/tensorflow1.jpg" width = 70%>
</div>