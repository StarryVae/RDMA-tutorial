# Using RDMA Efficiently for Key-Value Services

HERD这篇sigcomm 2014的文章，在个人看来，对于初学者来说是RDMA与具体应用结合方面最值得看的一篇文章，它的作者Anuj Kalia在RDMA领域发表了大量的A类会议系统性相关的文章，而且实验源码基本都是开放的，很多细节性的问题在github或者邮件上问他，他都会回复。之所以说这篇文章最值得看，关键在于它针对一个比较简单的应用key-value服务，进行了非常详细的RDMA参数选择的分析。一方面key-value这种应用相比于spark、tensorflow这些大型的系统更为简单，不需要花费大量的时间去了解系统的架构，而能更快的集中在RDMA与应用的结合方面；另一方面，这篇文章涉及到了RDMA verb、connection type等参数的选择，在理论上分析为什么这些参数对key-value性能会有影响，并且在实验中验证了参数选择的效果，在直观上感受到了RDMA参数选择的差异会给应用带来性能上的差异。

## RDMA参数选择

### WRITE or READ？

在这篇文章之前也有相关文章研究过RDMA加速key-value应用，比如pliaf，这些文章普遍使用READ verb实现GET的操作，因为READ不需要远端CPU的参与，使得key-value服务的GET操作就像在本地执行一样。但是，由于hash冲突的原因，GET操作平均需要2次以上的READ，也就是需要多次的round trip时间，而经过实验的对比，WRITE的延时是READ的一半，吞吐量也比READ高，因此如果使用WRITE发送GET或者SET操作，经由远端计算好后将结果返回，加起来只需要一次的round trip，性能会得到提升。

### Scalability

WRITE的性能这么高，那么是不是服务端返回结果时也可以用WRITE呢？不是。因为key-value这种应用是典型的一对多的传输模式，服务端需要和多个客户端建立连接，并且需要同时给多个客户端返回数据，那么网卡中就得保存大量的QP信息以及WQE的信息，而RDMA网卡的缓存是有限的，当缓存溢出时会从主机内存中取相应的WQE信息，性能会急剧下降。为了解决这个问题，这篇文章提出用UD的连接类型来进行服务端到客户端数据的传输，因为UD支持QP一对多的传输，服务端也就不需要创建过多的QP，从而提高可扩展性，具体的可扩展性对比如下图所示。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/HERD1.jpg" width = 70%>
</div>

## HERD的实现

这里我主要关注RDMA数据传输的实现，至于key-value的具体实现，文章也是借用的现有的一种key-value存储结构，就不做过多的叙述。而RDMA数据传输的实现难点主要是在利用WRITE将request发送给服务端，因为WRITE不需要服务端post recv的wqe，也就不会产生cqe，服务端也就无法感知请求的到来。为了解决这个问题，文章在应用层方面定义了需要传输的请求的格式，并且通过CPU去轮询其中的keyhash字段来判断是否有新的请求到来。简单来说，我们要想知道请求什么时候到来，只能通过监测接收请求的那块内存得知，那监测内存的什么区域呢？也就是需要自己去制定内存的组织形式，如下图，value、len、key就是内存组织的形式，CPU通过轮询key字段判断新的请求的到来，其实类似于自己实现了busy poll的功能。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/HERD2.png" width = 70%>
</div>

这篇文章以及Anuj Kalia后面的一些文章让我真正意义上认识并且体会到RDMA参数的选择对应用性能的影响，并且自己也通过RDMA编程去变换参数观察各个参数选择之后的性能，这也就为后续研究的第一个点打下了一些基础。现在再看这篇文章，实现方面提到的WRITE无法感知请求的问题，其实基本就是通过CPU去poll内存，很容易想到和实现，因为后面我把tensorflow改成WRITE时自己也是这么做的，后来才反应过来HERD也是这样。但是，现在来看是不是WRITE加上这种poll内存的方式在任何情况下都要比READ好呢？其实并不是，WRITE的这种方式很明显需要消耗大量的CPU，那么在数据中心多应用的场景下，有时需要大量的CPU资源用于计算，在CPU资源不足的情况下，依然使用WRITE这种方式的话，应用的整体性能可能还不如其他节省CPU的verb！

最后，自己看的很多文章都是类似于这种系统性的文章，但是实验室的科研更多强调理论，建模之类的，所以自己也是一直很困惑，是不是一定要固定模式的建模才能发好文章？这也是自己更想出去工作的原因，不适合科研，适合做苦力，修福报OvO