# Design Guidelines for High Performance RDMA Systems

Anuj Kalia对RDMA的研究越来越底层，这篇文章的Design Guidelines不是简单的连接类型、元语选择的guideline，而是从PCIe和网卡结构方面考虑影响RDMA性能的因素，最后反映到和具体应用结合时的一些设计细节，为本来的RDMA应用进一步带来性能上的提升。那我们具体来看文章中提到的RDMA design guidelines：

## Reduce CPU-initiated MMIOs

## Reduce NIC-initiated DMAs

## Engage multiple NIC PUs

## Avoid contention among NIC PUs

## Avoid NIC cache misses

以前看这篇文章的时候很蠢，只能get到Reduce NIC-initiated DMAs中提到的inline和unsignal，并且把这两个参数放入到了自己第一个研究点的框架当中，而其中涉及的batching、multiple NIC PUs的优化因为看不懂也没在意。现在看来，这些机制对RDMA性能有更大的提升！Anuj Kalia这些研究工作对真实系统有性能上的提升才是真正有意义的研究工作！