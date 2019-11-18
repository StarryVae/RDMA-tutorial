# Improving the Performance of Distributed TensorFlow with RDMA

这篇文章相比于HERD来说就显得简单粗暴一点，直接用RDMA的WRITE和READ替换了tensorflow的gRPC通信框架，并取得了极大的性能提升。当然，tensorflow这种深度学习框架本身就很复杂，能够将它与RDMA相结合在我看来本身就是挺有挑战性的一件事，更不用说如果像HERD一样去进一步分析tensorflow的特性，选择最佳的RDMA参数。

这篇文章的背景就是针对现在GPU设备的出现，集群计算能力增强的情况下，在进行分布式深度学习训练的时候，传统网络成为了较大的瓶颈。并且在TCP/IP环境下利用tensorflow进行了Alexnet模型的分布式训练实验，发现分布式训练的时间反而比单机训练时间长，这也就证明了TCP/IP网络带来的瓶颈。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/tensorflow2.png" width = 70%>
</div>

那么为了解决这个问题，文章的目标就是利用RDMA替换tensorflow基于TCP/IP的gRPC通信框架。首先是分析了tensorflow的send/receive流程，如下图所示，确定需要替换的模块。接着就分别利用RDMA的WRITE以及READ元语进行替换。最后测试性能的提升。整个文字的介绍过程很简单，但工程量应该是挺大的。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/tensorflow1.png" width = 70%>
</div>

当时在看这篇文章的时候，也去看了tensorflow的源码，发现有雅虎开源的RDMA的版本：https://github.com/tensorflow/tensorflow/tree/r2.0/tensorflow/contrib/verbs，源码也看了一段时间，因为当时是准备在RDMA与深度学习框架结合方面开展研究的。当时主要的想法也就是把雅虎的write_with_imm改成write以及考虑ps和worker一对多的传输是否也会有可扩展性的问题，后续也根据自己的想法修改过源码，编译，运行，记得是write成功替换了write_with_imm，性能反应在手写数字识别这个简单模型上有些许的提升。至于可扩展性的问题，一方面是现在深度学习集群很小，毕竟GPU这种设备很贵，另一面是UD只支持MTU的数据包传输，而深度学习模型中的参数会很大，超过MTU后在编程实现方面好像是很麻烦，因此后续也没有完成替换。最后因为这些方面也只是工程上的问题，没有理论科学问题，老师不是很认可，也就放弃了。

现在再看RDMA与tensorflow结合的这段经验，一是让我了解了RDMA与这种大型框架结合的基本的编程架构，另一方面，也对我后续研究的第一个点有些帮助，因为当时是在CPU集群中进行的实验，我用write替换write_with_imm后，因为需要监测数据请求的到来（见HERD的实现分析），CPU消耗变大，那后来就思考过，如果在CPU资源大量用来计算的场景下，write会不会还比write_with_imm好呢？答案是不，因为CPU资源不足的情况下，计算能力也受限，再把CPU资源消耗在网络传输上的话，是得不偿失的！