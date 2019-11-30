# LITE Kernel RDMA Support for Datacenter Applications

LITE发表在SOSP 2017，是我目前看过的最复杂的系统性文章，从题目也可以看到Kernel这个词，涉及到内核的编程应该不会很容易。阿里师兄APNet那篇文章据说灵感来自于这篇文章，我后续的很多研究也是受到这批文章的启发。

## 背景问题

这篇文章也是在RDMA与数据中心应用结合的场景下讨论目前RDMA还存在的问题：

* 缺少更为抽象易用的编程接口
* 因为MR的lkey、rkey、PTE数量以及QP的数量带来的RDMA可扩展性问题
* 缺少RNIC中资源的共享以及隔离机制

## 方案及挑战

就和之前为了解决内存、硬盘这些硬件存在的问题，而产生的虚拟内存系统和文件系统一样，文章觉得要解决RDMA硬件的问题也需要在内核实现一个管理RDMA资源的indirection layer。但是RDMA的优势得益于kernel bypass，那么文章需要面临以下挑战：

* 首先最大的挑战是如何在增加了这个indirection layer的情况下保证RDMA的性能
* 其次是如何在保证LITE性能的同时实现LITE的通用性和灵活性
* 最后是如何在不修改现有硬件，驱动以及操作系统的前提下实现LITE

## 系统设计

### LITE的内存抽象及管理

LITE的内存抽象主要考虑到LITE的通用及灵活性，因为原生RDMA需要应用自己注册内存等操作，比较麻烦，LITE在kernel的虚拟层中实现了自己的内存抽象及管理，对上层应用提供一些简单的内存分配、映射的接口；另外还考虑到RDMA的可扩展性问题，它将RNIC中的address mapping以及permission checking的功能实现在自己的内存管理中，并且利用RDMA提供的API将整个物理内存注册成一个MR，从而消除了RNIC中的PTE，减少了lkey、rkey的数量（变成1）。

### LITE RPC

LITE RPC是基于WRITE with imm元语实现的RPC机制，并且所有的RPC请求都共享一个CQ，节省CPU开销。还有一些用户态到内核态的优化可以具体看文章。

### LITE资源共享

LITE的资源共享指的是在应用之间实现QP、CQ的资源共享，但没有具体的资源共享的策略。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/LITE.png" width = 70%>
</div>

总的来说，LITE在kernel实现了一个虚拟层完成RDMA内存的管理、资源的共享、接口的抽象，并通过一系列系统性的优化在保证RDMA性能的同时，解决了RDMA在数据中心与应用结合存在的接口不匹配、可扩展性差、缺乏资源共享的问题。基于此进一步思考的话，其实LITE的接口抽象只是WRITE、READ、WRITE_WITM_IMM元语的封装，应用在与RDMA结合时还是要自己去决策选用哪种传输方式；另外LITE的资源共享只是系统性的实现，没有具体的资源共享策略，是否可以根据RDMA资源之间的特性找到应用之间资源共享的最佳策略。