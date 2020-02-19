.. Copyright by Kenneth Lee. 2020. All Right Reserved.

RoCE接口
========

RDMA
-----
RoCE是RDMA over Converged Ethernet的缩写，RDMA这个概念最早来自什么多个解决方案
，是谁先提出来的已经不容易确定了，我们也不关心这种复杂的历史发展过程，我们把关
注点放在这个特性的本质和在服务器领域的重要性上。

内存共享是分布式计算一个很关键的问题，如果计算节点比较近，我们可以直接基于总线
协议进行统一的内存共享，这是常见的SMP系统的解决方案。但如果总线的规模变大，很
多常见的总线访问接口就显得不合适了。比如说，一般的总线语义是不会认为某个访问对
象是会断开的，但如果通讯的规模变大，好比说，有十万个CPU，这些硬件就可能跨越很
多个机框、机架、甚至机房。这样就很难要求总线的节点都不会断开。这时，其实总线就
会接近于网络。

但专用的并行计算网络确实又不同于互联网，它不会经过复杂的，不可预测的公共网络，
轻易不会丢包。这样，一种中间的，比互联网可靠，但比总线稍差的互联需求就提出来了
。这催生了RDMA。

RDMA，Remote DMA本质上就是对着总线的DMA去的，它提供把一个节点的内存，同步到另
一个节点去的能力，同时，假设了这个同步过程是比本地总线拷贝慢的，可失败的，并且
很可能独立编址的。

RDMA有很多的实现方法和接口，但有一个影响比较大的标准组织：::

        http://www.rdmaconsortium.org/

为RDMA定义了很多基本的概念、原语和实现方法，成为很多RDMA方案的基础，其中包括我
们这里要介绍的RoCE协议。

RDMA Consortium的方案广泛用于各种数据中心，比如很多远程存储设备就使用RoCE接口
提供远程存储支持，……todo：需要更多例子细节。

todo：RoCE协议介绍。这样重点在于，RoCE和网络通讯的区别在于，RoCE的消息是直接写
到用户态需要的内存去的，而网络通讯是直接写到skb里面，要进一步的拷贝才能到用户
态。

todo：存储的使用场景要收集进来。


RDMA Consortium的基本概念空间
-----------------------------

todo：下面的逻辑是先堆上来，后面需要重新整理：

RDMA Consortium定义了一组实现无关的原语（Verb），基于这组原语，可以有不同的通
讯层和实现。

RDMA Constortium原语的使用者通过Queue Pair收发原语，Queue Pair是一对Queue，包
括一个Send Queue和一个Receive Queue（或者Share Receive Queue），还有一个辅助性
的Complete Queue。发出的原语称为Work Request，WR写到QP中，称为一个WQE，Work
Queue Element。Work Request发到Queue里面称为一个Workd Queue Element。

RDMA Constortium原语包括三类：

* 收发：Send （可选SE和Invalidate），Receive

* RDMA：Read（可选Invalidate），Write

* Memory：Bind，Fast Register MR，Invalidate Locale STag

这个原语包括如下基本概念：

* MR, Memory Registration
* MW, Memory Window
* ……

RoCE的软件支持
--------------
todo: RoCE用户态库OFED主线的位置和维护情况。


HNS3 RoCE实现
-------------

todo

.. vim: fo+=mM tw=78
