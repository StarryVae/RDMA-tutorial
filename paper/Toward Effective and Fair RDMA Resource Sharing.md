# Toward Effective and Fair RDMA Resource Sharing

这篇文章是APNet 2018的best paper，APNet虽然是近期才开始举办的一个会议，但是我去过两次，一次是中国的研讨会，一次是2019的正会，个人感觉学术氛围非常浓厚，大家很愿意一起交流，讨论，甚至质疑，也是因为在APNet会议上我和很多人对我poster的讨论让我对自己的工作和水平有了一个全新的认识，让我在面试过程中能更自信的表达自己。扯得有点远，文章的作者是我在阿里的师兄之前在南大的时候发的，因为当时我对这篇文章特别感兴趣，而碰巧的是我在github上关注的一个人好像就是他。。于是邮箱联系了师兄，师兄也是很快回复了，后来加了联系方式，师兄很热心的帮我解答各种问题，而且我们交流的过程中，讨论到看过的文章、作者之类的，基本是重叠的，让我感觉终于有人和我一样在做一个方向的研究了，非常有亲切感。好像又扯远了。。。

**进入正题**，文章的背景就是之前很多文章提到的RDMA网卡缓存有限导致的可扩展性的问题，现有研究都是直接通过多个线程共享QP来缓解这个问题，但是线程间需要竞争保护QP的锁，而这种锁竞争带来的开销会在线程增加的情况下极大的降低系统吞吐量；另一方面，在这种资源共享环境下，需要细粒度的调度来实现应用之间基于优先级的公平的资源共享。接下来就这两方面仔细看一下文章是怎么实现的。

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Avatar1.jpg" width = 70%>
</div>

<div align=center>
    <img src="https://github.com/StarryVae/RDMA-tutorial/blob/master/image/paper/Avatar2.jpg" width = 70%>
</div>

## 实现无锁的资源共享

## 实现公平的资源共享