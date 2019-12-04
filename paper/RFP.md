# RFP: When RPC is Faster than Server-Bypass with RDMA

## 背景问题

现有工作将RDMA与应用结合无非两种方式，一种是直接使用server-reply的方式替换socket的send/recv，这种方式实现简单，但是性能不如one-sided的verb；另一种则是利用one-sided verb替换socket获得更好的性能，但是需要针对具体应用设计复杂的数据结构或者协议。因此，本文的目的就是希望实现一个redesign cost小且性能更好的RPC机制。

## 解决方案

redesign cost通过RPC提供的接口很容易解决，问题是如何提供高性能的RPC接口，文章有两个重要的发现：

* RDMA inbound性能比outbound性能好，可以用来提高server reply的性能
* server-bypass的实际性能由于可能需要多次access完成一个请求的原因并不如预想的好

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/RFP1.png" width = 70%>
</div>

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/RFP2.png" width = 70%>
</div>

因此，基于以上两点发现，文章考虑RPC内部机制的实现如下：

* server端将请求的结果缓存在自己的内存中，而不是直接outbound到client，需要有client READ缓存在server内存的结果，这样就充分利用了inbound的性能优势
* server端需要参与处理client发过来的请求，这样就避免了server-bypass带来的多次access问题

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/RFP3.png" width = 70%>
</div>