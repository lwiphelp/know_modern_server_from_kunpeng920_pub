.. Copyright by Kenneth Lee. 2020. All Right Reserved.

Cache
======

介绍
----
相比访问本地资源，比如本地的寄存器，访问总线要慢得多。所以，作为一种自然的想法
，总线Station背后的每个独立实体，都会使用自己的存储系统，把本地运行的存储需求放
在本地，而把最终的输出再一次送出去。这是构架设计中常用的高内聚，低耦合的策略。

        | 高内聚，低耦合
        | 高内聚低耦合是构架设计的一种基本策略。
        | 计算机系统由大量的逻辑单元组成，
        | 为了理解和管理的方便，构架师和设计师会把这些单元分解到不同的模块中，
        | 而划分的原则就是，把关联密切的单元放在一个模块内部，
        | 这样，模块这一层的关联就少，这样，如果替换一个模块，
        | 这个模块需要遵守的约束就少，更容易维护。
        | 这一原则体现在物理实现中，高内聚的模块运行在同一个物理实体上，
        | 通讯减少，就能带来更好的性能。

.. figure:: high_cohesion_low_coupling.svg

但总线的高层语义却天然是一个高耦合的接口，因为每个地址，都可以成为一个关联。约
束这种耦合的方法是分层和分类，不同层次和分类关注不同的细节，这样复杂的关联在每
个层次和分类上就简单了。这本质还是实现高内聚，低耦合的模块。只是这时模块不是基
于功能分解的，而是基于语义空间进行分解的，这是另一个维度。

.. figure:: layered_classed.svg

Cache既是一种分层也是一种分类的而优化手段。这是从架构设计角度看Cache，从实现的
角度看Cache，不少设计者会认为Cache的存在是因为存储介质的成本原因。DDR的成本远远
低于速度更快的入SRAM等介质，这是在CPU中引入的Cache的一个重要原因。这恰恰是架构
设计的特点：每个决策都是一个多个多维度决策要素共同决定的。但决定只能有一个选择
，我们只能简单说某种情况我们做了什么样的选择，但我们很多时候清楚说明这种选择是
由那些少数的原因决定的。我们只能我我们尽量去找到其中的主要矛盾，或者说主要的控
制要素。

作为一种基本的优化策略，服务器系统中，Cache无处不在，CPU执行的指令可以有Cache，
CPU和设备访问的数据可以有Cache，MMU，IOMMU进行地址翻译可以有Cache。这些Cache有
些属于共同的地址空间，有些属于不同的地址空间。

Cache存在的前提是被访问的地址空间具有局部性，如果这个前提不存在，Cache就无法成
为帮助，而成为负担了。比如从磁盘中读出一组数据到内存，然后发给网卡。磁盘的数据
到达CPU，放在CPU的Cache中，CPU要求网卡去读这组数据，网卡对于CPU来说在总线的另一
侧，把数据放在CPU的Cache中就是个多余的操作。这时Cache减少关联的作用就不存在了。


内存Cache
----------
Cache有很多，每种起到不同的作用，但我们谈总线的时候，常常重点谈的是内存Cache。
内存Cache用于在更近的距离上提供更快的内存访问速度。

下图是鲲鹏920的内存Cache布置示意图：

.. figure:: kp920_cache.svg

这个Cache系统分成三个部分，在总线CPU一侧，鲲鹏920的CPU核用了两级的Cache，这更多
是从成本上考虑的，其中的L1指令和数据分开，分别64K。L2部分，它大很多，512K。

todo：L1和L2的成本差距，（这个数据讲在正式版本才有可能提供）

而第三级的Cache，主体其实是在另一个Station上的，主要用于给一个Ring内的用户提供
缓存，CPU只保留L3 Cache的Tag，以便知道数据在哪里，要取这个数据，需要穿过Ring的
总线，到另一端去取。

而数据的本体，在DDR上，DDR通过DDRC连接，接到片外的DIMM条上。鲲鹏920的DDR控制器
连接在SCCL的一个独立的Station上，每个SCCL上有4个DDRC，每个DDRC可以连一到两个
DIMM插槽（能连多少个取决于内存的主频和工程工艺问题）。

        | DDR
        | todo

        | DIMM
        | todo

基于这个结构，当CPU发出一个内存读写请求，通过L1，L2，L3 Cache层层代理，不一定会
立即反映为内存的读写。这个结构，一方面提高了效率，但另一方面，增加代理就会增加
复杂度，后面我们会初步看看这种复杂度如何反映到软件的语义空间，更详细的讨论，我
们会放到计算子系统中。

Cacheline
```````````
CPU进行内存读写，可以有不同的读写长度，而Cache的读写长度是固定的。这样设计作者
认为主要是这样的原因：

1. 电路设计简单
2. 内存访问具有局部性，一次成片进行访问是有利的博弈
3. 节省地址

Cache远远小于内存，所以它是内存的一个离散分布记录，它需要为每个数据记录保存一个
地址。如果每个地址代表的空间的大小，Cache的空间就大部分被地址所利用了。所以
Cache的访问长度总是固定，而且比较长的，这个长度就称为Cache的Cacheline。

不同级别的Cache的Cacheline长度不同，取决于业务模型和参数的差异。鲲鹏920的L1，L2
的Cache长度是64位的，L3是128位的。

对于程序员来说，让数据结构尽量Cacheline对齐，这对性能会有帮助。但具体对齐那个长
度，会不一样。一般来说，按L1对齐是比较安全的。

在Linux下，可以通过下面命令读到这个长度：::

        getconf LEVEL1_DCACHE_LINESIZE获得这个长度

在指令一级，ARMv8架构定义ctr_el0提供Cacheline的长度，鲲鹏920上可以通过读这个寄
存器获得这个长度。

.. code-block:: c

        asm volatile ("mrs\t%0, ctr_el0" : "=r" (ctr_value))        

todo: 寄存器格式和代码运行结果

Cacheline的地址有两种可能的设计，一种是基于虚拟地址，一种是基于物理地址。一般用
物理地址比较常见。在鲲鹏920上，除了L1指令Cache，都是基于物理地址寻址的。而指令
Cache比较特别，它有时基于物理地址，有时基于虚拟地址，这个取决于指令译码器的行为
，如果指令译码器同时给Cache和MMU发请求，第一次发出去的时候，肯定是个虚拟地址（
因为MMU还没有响应），如果查到了，就用虚拟地址寻址了。如果没有查到，但MMU返回了
物理地址，译码器就会用物理地址来寻址。

Cache预取
`````````
Cache Prefetch也是一个针对Cache的优化设计。Cache比实际的内存快很多，所以如果我
们可以提前加载部分内存到Cache中，就会在性能上有优势。

鲲鹏920通过ARMv8的PRFM系列指令实现预取准备。PRFM系列指令是一组内存系统的Hint指
令，在功能上它可以认为相当于一个nop（空操作）指令，但总线的内存相关功能可以通过
这个通知提前对Cache进行准备，可能包括从内存上读入内容，也可能提前在Cache上分配
空间，（有些SoC实现甚至可以什么都不干，反正这是个实现相关的功能）这个动作可以和
其他操作并行发生，考虑到一般的指令只需要几个时钟周期，而读写一次内存需要上百个
时钟周期，这个并行就能带来很多明显的优势。

下面是Linux内核中鲲鹏HNS3网卡驱动通过预取进行数据收发的代码：

.. code-block:: c

        netdev_tx_t hns3_nic_net_xmit(struct sk_buff *skb, struct net_device *netdev)
        {
                struct hns3_nic_priv *priv = netdev_priv(netdev);
                struct hns3_enet_ring *ring = &priv->ring[skb->queue_mapping];
                struct netdev_queue *dev_queue;
                int pre_ntu, next_to_use_head;
                struct sk_buff *frag_skb;
                int bd_num = 0;
                int ret;

                /* Prefetch the data used later */
                prefetch(skb->data);

                ret = hns3_nic_maybe_stop_tx(ring, netdev, skb);
                if (unlikely(ret <= 0)) {
                        if (ret == -EBUSY) {
                                u64_stats_update_begin(&ring->syncp);
                                ring->stats.tx_busy++;
                                u64_stats_update_end(&ring->syncp);
                                return NETDEV_TX_BUSY;
                        } else if (ret == -ENOMEM) {
                                u64_stats_update_begin(&ring->syncp);
                                ring->stats.sw_err_cnt++;
                                u64_stats_update_end(&ring->syncp);
                        }

                        hns3_rl_err(netdev, "xmit error: %d!\n", ret);
                        goto out_err_tx_ok;
                }

                next_to_use_head = ring->next_to_use;

                ret = hns3_fill_skb_to_desc(ring, skb, DESC_TYPE_SKB);
                if (unlikely(ret < 0))
                        goto fill_err;
        ...
        }

这是skb网络数据发送的代码，最前面的perfetch(skb->data)本身不产生功能，但去掉这
一行，网卡的性能就会有明显的下降：

todo：需要一个删除prefetch的性能数据。

Cache Coherency
================

Cache制造了多份数据，这又会带来数据同步的问题。比如，总线用户A有Cache，它修
改了某个地址的内容，这个修改暂存在A本地的Cache中。然后总线用户B要来读这个数据，
它怎么知道这个最新的数据在A的Cache中？

这种问题仍有透明和不透明两种设计。不透明的设计，要求用户自己知道Cache的存在，如
果要通知其他总线用户，就必须主动进行刷新，广播等等。而透明是说，总线有机制保证
知道Cache中有数据被修改了，它总能保证每个总线用户都是知道什么数据在Cache中，并
有办法得到最新的数据的。这种Cache特性，称为Cache Coherency，简称CC。

ARMv8架构要求所有SMP的CPU在Inner域中，必须是互相是Cache Coherency的，对于设备则
没有要求。而鲲鹏920使用全CC总线，所有的CPU，加速器，设备都是CC的，不需要使用者对
Cache做任何特殊处理。

CC可以有多种机制实现，鲲鹏920主要通过Snooping实现。Snooper跟踪共享的地址的
Cacheline，如果发生更改，就通过总线消息通知所有的用户同步消息。所以，共享方越多
，这个协议的效率越低。如果没有Cacheline共享，Snooper不会工作，性能不会有任何影
响，但如果有很多方共享同一个数据，这个效率就会掉下去。这种情况常常发生在
spinlock的情形下。比如你有32个核参与计算，你使用spinlock，那么每次有一个核更新
了spinlock，snooper就要通知31个核这个数据发生了更新，这个效率会变得非常低。特别
是由于总线是一个去中心化的系统，并没有一个控制中心控制一个全局的行为，每个用户
发现自己的Cache被刷新了，就想要去通知其他方，这样会导致互相更新对方，如果发生冲
突，这个性能就会进一步下降。

todo：需要一副Snooper工作原理的图

        | Spinlock
        | Spinlock，中文常翻译为自旋锁，是一种常用的共享内存多核系统的同步手段。
        | 其原理是所有需要同步的CPU等待一个相同的内存地址的内容转变为特定的值，
        | 才进入互斥的代码中访问公共资源。
        | Spinlock通常需要CAS指令的支持。

        | CAS
        | Compare-And-Set指令是一种原子指令，
        | 可以全局原子化地判断某个内存地址的内容，
        | 并在内容变成特定的值的时候，把它设置为指定的值。
        | 这个过程对于所有的其他核来说都是原子的，
        | 也就是说，对于这些核来说，Compare和Set两个操作或者同时生效，
        | 或者都不生效。

解决这个问题的方法是减少数据的共享方。Linux中mcs_spinlock（封装为qspinlock），
就是为解决这个问题而引入的。Mcs_spinlock的原理图示如下：

        .. figure:: mcs_lock.svg

它是一种典型的空间换时间的设计。每个新的等待者进入等待了，不等在原来的锁上，
而是等待在一个新分配的共享变量上，一旦前一个等待者拿到锁了，这个等待者就开始通
过新的共享变量和下一个等待者互相等待了。这样同一个地址上的等待者就会减少。但很
明显，这增加了内存和准备时间。

Linux通过如下配置项使能qspinlock功能，在ARM平台上，这个配置是默认开启的。

::
        CONFIG_ARCH_USE_QUEUED_SPINLOCKS=y
        CONFIG_QUEUED_SPINLOCKS=y
        CONFIG_ARCH_USE_QUEUED_RWLOCKS=y
        CONFIG_QUEUED_RWLOCKS=y

