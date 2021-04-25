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
* Direct：内核旁路技术、协议栈下放到网卡
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

另外，由于RDMA将协议栈卸载到了网卡中，因此RNIC中也实现了相应的网络协议，以infiniband为例，它的协议栈也是分层的

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA协议栈.png" width = 70%>
</div>

## 二、RDMA编程

在了解了RDMA架构之后，只知道它能提供更低的延时和更高的带宽，但没有去亲手验证它的低延时和高带宽，就总感觉虚无缥缈。后来通过各种渠道搜索发现有个perftest工具，可以用来测试RDMA的性能，举个例子：

```
server：sudo ib_send_lat -F -s 10 -n 1000
client：sudo ib_send_lat 10.128.16.214 -F -s 10 -n 1000
```

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/perftest结果.png" width = 70%>
</div>

这个例子可以看到10B大小的数据包延时在1.5us左右，后续去分析perftest的源码的话，就会涉及到RDMA编程的细节。

### 2.1	RDMA数据传输

介绍具体RDMA编程前，先以类似于传统socket编程的send/recv为例，了解一下RDMA微观的数据传输过程，如图4所示，主要分为以下几步：

* 创建QP、CQ，建立收发两端QP之间的连接（类似于socket功能）；
* 接收端注册用户内存到网卡RNIC，并发起接收的请求，该请求中就包括了注册到网卡的内存地址和长度；
* 发送端注册用户内存到网卡RNIC，并发起发送的请求，该请求中就包括了注册到网卡的内存地址和长度；
* 发送端网卡执行发送请求，根据发送队列SQ中发送请求的内存地址和长度到用户态内存中直接读取数据发送给接收端；
* 当数据到达接收端时，接收端网卡执行接收请求，根据接收队列RQ中接收请求的内存地址和长度将接收的数据直接写到相应的位置；
* 接收端数据接收完成后产生一个完成通知CQE到完成队列CQ中，程序从完成队列中取出完成通知CQE代表整个传输过程的结束。

上述文字描述过程看上去很简单，和socket编程差不多，但是由于RDMA编程库的封装性不强以及在整个传输过程中涉及大量的参数选择，RDMA编程其实极为复杂。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA send微观传输图.jpg">
</div>

### 2.2 RDMA参数

前面说RDMA在整个传输过程中涉及大量的参数选择，如图5所示，标注了整个过程涉及的主要参数：

* QP类型：RC、UC、UD（R: reliable, U: unreliable, C: connection, D: datagram），QP的类型需要在建立连接时确定，就像在建立socket通信时，需要确定是TCP还是UDP的连接类型。其中，R、U的区别在于是否是可靠传输，也就是是否返回ack，C、D的区别在于是否是面向连接的，C指的是面向连接的，类似于TCP，在数据传输前，会先建立好连接，也就是QP互相交换相应信息，而D指的是datagram，QP之间连接的信息不是事先确定好的，而是放在数据包的包头，由数据包决定所要发送的具体的接收端的QP是哪个；
* Verb：send/recv、write、read，具体的传输数据的方式，send/recv和socket类似，write指的是将数据从本地直接写到远端内存，不需要远端CPU参与，read指的是将数据从远端直接读到本地，同样不需要远端CPU参与；
* Inline/non-inline：inline在C++里指的是内联，在程序编译时直接用函数代码替换函数调用，节省时间，但针对的是那些执行时间较短的函数。同样，在RDMA里面，inline指的就是将一些小的数据包内联在发送请求中，这样在2.1中RDMA数据传输的第四步，就可以少一次取数据的过程；
* Signal/unsignal：signal/unsignal指的是是否产生CQE，如果使用unsignal，将不会产生CQE，因此也就不需要poll CQE，从而减少CPU使用，提高延时。但是，如果使用了unsignal，必须保证隔一段时间发送一个signal的请求，因为如果没有CQE的产生以及poll，那么这些unsignal的发送请求将一直占用着发送队列，当发送队列满时，将不能再post新的请求；
* Poll策略：poll策略指的是poll CQE的方式，包括busy polling、event-triggered polling，busy polling以高CPU使用代价换取更快的CQE poll速度，event-triggered polling则相应的poll速度慢，但CPU使用代价低；

在了解了RDMA各种参数后，接下来将介绍具体的RDMA编程中，这些参数是如何体现的（在具体参数选择出会用黄色背景标注出）。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA传输参数图.png">
</div>

### 2.3 send/recv

这里send/recv指的就是上述verb的类型，因为不同verb的通信过程还是有一定区别的，因此分开介绍。那么send/recv在代码的哪里指定呢？我们首先按照2.1中RDMA数据传输的步骤一步一步来：

#### 2.3.1 建立连接

为了简便建立连接的过程，我们利用rdmacm库提供的一系列接口来实现。值得注意的是QP的创建必须在rdma_connect以及rdma_accept之前，因为这两个函数主要封装了QP信息的交换，而如果不使用rdmacm库的话，自己也可以利用socket写一套交换QP信息的程序。

Server端：

* 监听连接：

```
ec = rdma_create_event_channel();
rdma_create_id(ec, &listener, NULL, RDMA_PS_TCP);
rdma_bind_addr(listener, (struct sockaddr *)&addr);
rdma_listen(listener, 10); 
```

* 创建QP：

```
qp_attr->send_cq = s_ctx->cq;
qp_attr->recv_cq = s_ctx->cq;
qp_attr->qp_type = IBV_QPT_RC;

qp_attr->cap.max_send_wr = 10;
qp_attr->cap.max_recv_wr = 10;
qp_attr->cap.max_send_sge = 1;
qp_attr->cap.max_recv_sge = 1;

rdma_create_qp(id, s_ctx->pd, &qp_attr)；这里要注意的是qp_attr，顾名思义，qp的属性，那么qp的类型也就是在这个结构中指定，以RC连接类型为例：qp_attr->qp_type = IBV_QPT_RC;
```

* 完成连接：

```
rdma_accept(id, &cm_params);
```

Client端：

* 解析地址、路由：

```
ec = rdma_create_event_channel();
rdma_create_id(ec, &conn, NULL, RDMA_PS_TCP);
rdma_resolve_addr(conn, NULL, addr->ai_addr, TIMEOUT_IN_MS);
rdma_resolve_route(id, TIMEOUT_IN_MS);
```

* 创建QP：

```
参考server端；
```

* 发起连接：

```
rdma_connect(id, &cm_params);
```

#### 2.3.2 注册内存

```
ibv_reg_mr( s_ctx->pd, conn->send_region, BUFFER_SIZE,IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE) );
```

其中，第二和第三个参数分别代表用户态内存地址和长度；最后一个参数指的是访问该内存的权限：IBV_ACCESS_LOCAL_WRITE指的是允许本机的写权限、IBV_ACCESS_REMOTE_WRITE指的是允许远端的写权限、本地的读权限是默认的、IBV_ACCESS_REMOTE_READ指的是允许远端的读权限。

#### 2.3.3 发起接收请求

前面步骤中介绍了接收请求主要包括了注册到网卡的内存地址和长度，那么体现在代码中就是wr（接收请求）的sg_list成员，指定完内存后，ibv_post_recv函数就完成了将请求发送给网卡接收队列的任务。其中第一个参数指的就是网卡中的具体QP。

```
wr.sg_list = &sge;

sge.addr = (uintptr_t)conn->recv_region;
sge.length = BUFFER_SIZE;
sge.lkey = conn->recv_mr->lkey;

ibv_post_recv(conn->qp, &wr, &bad_wr);
```

#### 2.3.4 发起发送请求

指定内存的部分和接收请求一样，发送请求比接收请求主要多了三个参数的设置，也就是前面介绍参数时说的verb、Inline/non-inline以及Signal/unsignal。

```
wr.opcode = IBV_WR_SEND;
wr.send_flags = IBV_SEND_SIGNALED | IBV_SEND_INLINE;
wr.sg_list = &sge;

sge.addr = (uintptr_t)conn->send_region;
sge.length = BUFFER_SIZE;
sge.lkey = conn->send_mr->lkey;

ibv_post_send(conn->qp, &wr, &bad_wr);
```

#### 2.3.5 从完成队列中poll完成通知

event-triggered polling指的就是需要下面前三行事件通知的代码，busy polling就不需要前三行代码，直接循环地poll cq，因此消耗更多的CPU。

```
void * poll_cq(void *ctx)
{
  struct ibv_cq *cq;
  struct ibv_wc wc;

  while (1) {
    TEST_NZ(ibv_get_cq_event(s_ctx->comp_channel, &cq, &ctx));
    ibv_ack_cq_events(cq, 1);
    TEST_NZ(ibv_req_notify_cq(cq, 0));

    while (ibv_poll_cq(cq, 1, &wc))
      on_completion(&wc);
  }

  return NULL;
}
```

### 2.4 write/read

在具体介绍write/read编程前，先以write为例看一下write的微观传输图，比较它和send/recv的区别。如图6所示，首先，write不需要在接收端发起接收请求，但是相应的需要将注册的内存地址和key值（代表写的权限）发送给发送端，然后发送端在发起发送请求时，就会包括接受端的这个内存地址和key值，直接将数据写到远端内存，而不需要接收端CPU参与。

那么接收端如何将自己注册的内存地址和key值发给发送端呢？很简单，既然我们上面已经掌握了send/recv的编程，那么就直接用send/recv发送，具体过程参考2.3，这里主要介绍一下发送端在接收到这些信息后，如何将本地数据写到远端，和send/recv相同的参数不再赘述，主要看第二、三行，分别是远端地址和key值，另外opcode变成了WRITE：

```
wr.opcode = IBV_WR_RDMA_WRITE;
wr.wr.rdma.remote_addr = (uintptr_t)M.remote_addr;
wr.wr.rdma.rkey = M.rkey;
ibv_post_send(conn->qp, &wr, &bad_wr);
```

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA write微观传输图.jpg">
</div>

### 2.5 code example

[rdma_send_recv](https://github.com/StarryVae/RDMA-tutorial/tree/master/example%20code/rdma_send_recv)

[rdma_write_read](https://github.com/StarryVae/RDMA-tutorial/tree/master/example%20code/rdma_write_read)

### 2.6 RDMA参数性能对比example

在熟悉了RDMA编程以及各个参数在代码中的具体位置后就可以设计大量的对比实验观察参数的不同对延时或吞吐量性能的影响，可以直接利用perftest工具，当然最好可以自己写测试代码。

#### 2.6.1 inline/non-inline

以send/recv为例比较inline/non-inline参数对延时性能的影响：

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/配置-inline.png" width = 70%>
</div>

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/实验结果-inline.png" width = 70%>
</div>

#### 2.6.2 verb

以write和read为例比较verb参数对延时性能的影响：

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/配置-verb.png" width = 70%>
</div>

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/实验结果-verb.png" width = 70%>
</div>

#### 2.6.2 poll strategy

以send/recv为例比较poll参数对延时性能的影响：

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/配置-poll.png" width = 70%>
</div>

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/实验结果-poll.png" width = 70%>
</div>

### 2.7 RDMA与应用的结合

在熟悉了RDMA编程以及RDMA各个参数的性能后，就可以将RDMA与具体的应用相结合以发挥RDMA的优势，提高应用的性能。如图7所示，现有应用主要包括：大数据应用，深度学习框架，分布式存储，分布式共享内存等。

基础的应用可以参考：HERD[1]、redis[2]等。

进阶的应用可以参考：tensorflow[3]、spark[4]、ceph[5]等。

需要注意的是在将RDMA与应用结合的同时，我们要考虑到如何充分利用RDMA的性能，也就是如何选择上述介绍的RDMA参数。大量单纯在工程上修改应用的工作只是将普遍认为的高性能的RDMA参数运用到应用通信框架中，比如利用one-sided的verb传输数据，利用busy polling获取完成通知。但是这种方式却忽略了应用自己的特性以及服务器的资源状态，具体的RDMA参数选择与不同应用的应用特性以及相关的服务器资源状态有关。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/RDMA应用部署图.png">
</div>

这里以tensorflow框架为例，拿最简单的手写数字识别的模型进行分布式训练，直观的感受一下RDMA对应用性能的提升

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/tensorflow实验.png" width = 70%>
</div>

## 三、RDMA相关研究

### 3.1	传输层/应用层

* 利用RDMA加速特定应用
* 易于使用且高性能的RDMA编程库[6]
* QP share解决可扩展性问题[7]
* 自动化工具判断各种应用是否适合rdma

### 3.2	网络层/链路层

* 拥塞控制[8][9]
* 多路径传输[10]

### 3.3	虚拟化

* RDMA+docker容器云[11]

## 四、读过的一些文章

这里我主要列举一些在我学习RDMA过程中看过的一些有意义，对自己提升较大的文章，并且详细介绍这些文章的亮点以及自己看完的收获。具体的内容移步https://github.com/StarryVae/RDMA-tutorial/tree/master/paper

## 五、面试相关

RDMA相关的面试经验：https://github.com/StarryVae/RDMA-tutorial/tree/master/interview

## 六、参考

1.	Using RDMA Efficiently for Key-Value Services.
2.	Accelerating Redis with RDMA Over InfiniBand.
3.	https://github.com/tensorflow/tensorflow/tree/v2.0.0/tensorflow/contrib/verbs
4.	https://github.com/Mellanox/SparkRDMA/blob/master/src/main/java/org/apache/spark/shuffle/rdma
5.	https://github.com/ceph/ceph/tree/master/src/msg/async/rdma
6.	RFP: When RPC is Faster than Server-Bypass with RDMA.
7.	Toward Effective and Fair RDMA Resource Sharing.
8.	Revisiting Network Support for RDMA.
9.	HPCC: High Precision Congestion Control.
10.	Multi-Path Transport for RDMA in Datacenters.
11.	FreeFlow: Software-based Virtual RDMA Networking for Containerized Clouds.


