.. Copyright by Kenneth Lee. 2020. All Right Reserved.

Ethernet接口
============

Ethernet
---------
以太网1980开始商用，并在1983年成为IEEE标准的一部分，定义为802.3协议族。
主要用于短距离通讯，比如LAN，MAN，WAN等场景。

最早的Ethernet实现采用同轴电缆，而现代人看得更多的是这样的的使用双绞线的版本：

todo：需要一张双绞线网口的图片（公母）。

而数据中心还有更常见的光纤：

todo：需要光纤网口的图片。

Ethernet的带宽也最初的2.94 Mbit/s发展到当前最高400 Gbit/s。

从ISO定义的七层网络模型来考量，Ethernet主要提供从物理层到数据链路层的服务。
Ethernet技术广泛用于TCP/IP协议栈的实现，为TCP/IP协议提供数据链路层服务。而后者
，基本上可以认为是现代互联网的基础。Ethernet使用一个6个字节的MAC地址表示一个通
讯地址，MAC地址由IEEE管理和分配，其中前三个字节称为OUI，Organizationally
Unique Identifier，用于成段分配给特定的Ethernet网卡生产商，由于每个分段的数量
有限，并非每个企业的分段唯一的，比如仅华为公司就可以占用数百个OUI。具体读者可
以从Internet上搜索到。用户也可以强行给自己的Ethernet网卡设置自己本地使用的MAC，
只要能保证使用范围之内没有冲突即可（当然，我们并不建议这样做）。

Ethernet协议包括数字的部分和模拟的部分，数字的部分负责接纳CPU的数据，封装数据
帧等。todo...

Linux Ethernet子系统
--------------------

Linux Ethernet子系统本身很简单，仅仅是对核心网络系统的封装。
它的实现在Linux内核源代码如下位置：::

        net/ethernet/

简单浏览一下这个目录的代码，可以发现它其实仅仅是一组Helper函数，只是把大部分
Ethernet驱动调用网络子系统（重点是对net_device数据结构的封装）都需要使用的逻辑
组合成公共的函数以供调用而已。所以我们分析Ethernet子系统，只要分析网络子系统的
行为就可以了。

Linux网络子系统
````````````````
在主线Linux v5.5中，网络设备子系统在如下目录定义：::

        net/

其中的核心代码在软件子目录中：::

        net/core/

这个核心代码主要是协议基本无关的Socket接口及其管理，网络设备管理等逻辑。而不同
的协议在net的其他目录中，而设备驱动程序则在drivers/net/目录中。

网络子系统是Linux设计非常复杂和而是是Linux内核发展最迅速的子系统之一，本书并不
打算对这个子系统进行深入的探讨，我们重点放在它和硬件的互动模型上。

Linux网络子系统为不同的硬件、协议提供到用户态的BSD Socket接口通道，我们可以这
样抽象它的基本结构：

        .. figure:: linux_net_subsystem_struct.svg

无论是发送还是接收方向，都有至少两个异步的执行流，所以天然就需要一个Buffer去同
步两个执行流的速度。

在发送方向，进程通过sock_send系列的接口，对外发送消息，而网卡驱动，需要等待网
卡设备准备就绪，才能把数据发出去，在这之间，数据需要缓冲在CPU中，或者说缓冲在
CPU控制的内存中。Linux的实现是把这个数据留在协议处理层中进行管理。

反过来，在接收方向，网络的数据就绪了，驱动必须把数据收走，但用户程序不一定准备
好接收数据，在这个间隙中，数据也需要缓冲在协议处理层中。

Linux使用一个称为sk_buff的数据结构作为统一的数据报载体，以便在内核之内共享所有
数据并避免拷贝。sk_buff包含很多细节，但其设计主要利用了两个设计技巧：其一是报
文头预留，也就是如果已经预计到这是什么协议，可以在数据的前面留下空间，后续的协
议层可以在这个数据前面补充数据头。另一个技巧是允许数据分段，这样如果后部数据不
足，可以通过增加分段不断补充数据。这样，在整个内核协议栈内部基本上不产生多余的
内存和拷贝的需要。

以上两个逻辑图示如下：

        .. figure:: linux_net_data_flow.svg

从内存拷贝的角度看，从进程到内核变成sk_buff，这会产生一次拷贝，之后都是协议栈
内部的修改sk_buff和指针调度的过程，最后数据从内存中进入设备，这会产生第二次拷
贝。对这两次拷贝还有优化的余地，这在后面讨论DPDK和HNS3的实现的时候我们再来重新
审视。

从执行的角度来看，无论有多少线程、进程上下文通过Socket接口进入网络子系统，都可
以有一个缓存的过程，在由调度程序等待网络设备就绪，然后发到网络上。反过来，设备
的数据就绪了，无论有多少接收线程（包括中断）把数据送入CPU，最终还是可以在协议
处理层进行缓存，等待上层的Socket应用程序把数据收到自己的空间中。

为了把接收和业务处理的压力分散到多个CPU上，操作系统和网卡间可以支持多队列协议
，其原理是把网卡的数据通道分成多个独立的子通道，从而可以把压力分流到不同的CPU
上。由于HNS3支持这个特性，我们在探讨HNS3的时候再讨论这个方案的细节。

这个业务模型可以在两个地方产生流控或者丢包。一个是分配sk_buff的时候，如果内存
耗尽了，就会发生丢包，另一个是在进入协议处理层的时候，不同的协议模块可以控制允
许多少数据被缓存，从而决定接受还是拒绝这个报文。对于100G的网络，这两点都尤其重
要，正如我们在本章开始的时候说到的，100G的网络意味着每秒12.5G的数据流量，只要
业务层或者网络稍有卡顿，都会产生大量的数据，而无限增加缓存又会导致更大的时延造
成更多的网络拥塞，这是进行网络业务调优需要考虑的关键问题。

网络设备和Linux网络子系统的关系主要通过net_device这个数据结构进行封装。设备驱
动程序通过register_netdev()注册协议栈需要的数据结构和回调函数，从而配合协议栈
对网络设备进行控制。register_netdev()成功以后，ifconfig -a或者ip link show命令
就可以看到这个设备，ifconfig up和ip link x up命令则会把设备投入运行。

net_device中可以设置ethtool_ops，这里可以为Ethernet设备设置设备控制回调，这个
设计从分层上不太合理，因为一般会认为这应该封装在Ethernet设备一层。实际上，在
Linux内核的代码中，net_device这个数据结构有这样一个注释：::

        Actually, this whole structure is a big mistake.  It mixes I/O
        data with strictly "high-level" data, and it has to know about
        almost every data structure used in the INET module.

这个说法有道理，但Linux内核网络协议栈做成这样也完全可以理解，关键在于Linux网络
子系统一直是以性能为中心进行优化的，所有行为都让位于数据流通道，如果太重视死板
的分层，它就失去竞争力了。用一个net_device作为协议栈和设备一个大箩筐式的接口中
心，也就成了可接受的设计了。

除了基本的分层协议处理，现代服务器网络设备上还实现了很多Offloading功能，尽量把
机械的工作下沉到网卡设备，比如IP_CSUM，GRO等。Linux实现这些功能的方法是在
net_device->xxx_feature域中标记硬件当前使能了哪些能力。如果这部分能力是能了，
软件就可以不用计算相关的协议的域（比如某些checksum），而交给硬件去完成，在
Linux内核的代码中可以看到HNS3支持的全部硬件加速功能：

.. code-block:: c

   // drivers/net/ethernet/hisilicon/hns3/hns3_enet.c:hns3_set_default_feature
   netdev->hw_enc_features |= NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM |
        NETIF_F_RXCSUM | NETIF_F_SG | NETIF_F_GSO |
        NETIF_F_GRO | NETIF_F_TSO | NETIF_F_TSO6 | NETIF_F_GSO_GRE |
        NETIF_F_GSO_GRE_CSUM | NETIF_F_GSO_UDP_TUNNEL |
        NETIF_F_GSO_UDP_TUNNEL_CSUM | NETIF_F_SCTP_CRC |
        NETIF_F_TSO_MANGLEID | NETIF_F_FRAGLIST;

   netdev->gso_partial_features |= NETIF_F_GSO_GRE_CSUM;

   netdev->features |= NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM |
        NETIF_F_HW_VLAN_CTAG_FILTER |
        NETIF_F_HW_VLAN_CTAG_TX | NETIF_F_HW_VLAN_CTAG_RX |
        NETIF_F_RXCSUM | NETIF_F_SG | NETIF_F_GSO |
        NETIF_F_GRO | NETIF_F_TSO | NETIF_F_TSO6 | NETIF_F_GSO_GRE |
        NETIF_F_GSO_GRE_CSUM | NETIF_F_GSO_UDP_TUNNEL |
        NETIF_F_GSO_UDP_TUNNEL_CSUM | NETIF_F_SCTP_CRC |
        NETIF_F_FRAGLIST;

   netdev->vlan_features |=
        NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM | NETIF_F_RXCSUM |
        NETIF_F_SG | NETIF_F_GSO | NETIF_F_GRO |
        NETIF_F_TSO | NETIF_F_TSO6 | NETIF_F_GSO_GRE |
        NETIF_F_GSO_GRE_CSUM | NETIF_F_GSO_UDP_TUNNEL |
        NETIF_F_GSO_UDP_TUNNEL_CSUM | NETIF_F_SCTP_CRC |
        NETIF_F_FRAGLIST;

   netdev->hw_features |= NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM |
        NETIF_F_HW_VLAN_CTAG_TX | NETIF_F_HW_VLAN_CTAG_RX |
        NETIF_F_RXCSUM | NETIF_F_SG | NETIF_F_GSO |
        NETIF_F_GRO | NETIF_F_TSO | NETIF_F_TSO6 | NETIF_F_GSO_GRE |
        NETIF_F_GSO_GRE_CSUM | NETIF_F_GSO_UDP_TUNNEL |
        NETIF_F_GSO_UDP_TUNNEL_CSUM | NETIF_F_SCTP_CRC |
        NETIF_F_FRAGLIST;

   if (pdev->revision >= 0x21) {
        netdev->hw_features |= NETIF_F_GRO_HW;
        netdev->features |= NETIF_F_GRO_HW;

        if (!(h->flags & HNAE3_SUPPORT_VF)) {
                netdev->hw_features |= NETIF_F_NTUPLE;
                netdev->features |= NETIF_F_NTUPLE;
        }
   }

我们在后面看HNS3的设计的时候再看一些具体的例子，看它们具体是如何工作的。

NAPI
````
Linux网络子系统的构架并不约束网络数据流的调度模型，但作为最佳实践，Linux的默认
网络接口调度模型是：在中断处理向量中启动网卡软中断，然后在网卡软中断中发送和接
收数据。

这种方法是基于中断模式较为顺理成章的设计。CPU访问设备的数据，一般只有两种选择
：

* 轮询模式：CPU定期去访问设备状态，发现设备就绪就开始收发。这比较适合设备数据
  比较密集或者虽然数据不密集，但对时间性要求不高的场合。缺点是可能不少多余的设
  备访问操作。

* 中断模式：CPU主要在做其他业务，设备就绪就通过中断通知CPU进行收发。这比较适合
  CPU处理其他业务和IO比较均衡的情况。好处是基本不会有多余的设备访问。缺点是如
  果数据密集，就会有很多多余的中断过程，而中断过程本身消耗CPU资源，因为需要备
  份多余的上下文。

作为通用服务器的方案，Linux内核采用后者作为一般网络设备的调度模型，同时，提供
NAPI接口，作为大部分高性能网卡的标准调度方法，它不是强制要求的，但使用这个API
可以简化驱动设计和优化调度过程。

NAPI是一种中断聚合的方案，试图综合轮询和中断两种方式的优势。它在网卡收到中断决
定调度后，关闭中断进行一段时间的持续轮询，从而提高收发的效率。

这样的调度方式常常要面对这样一个问题：如果设备中断告知设备就绪了，你一次接收或
者发送多少数据？如果你总是收发到上限，那么CPU会有很长一段时间都在收发上，而不
能处理这些数据，这样每波的数据缓冲可以很高，而且不一定值得。

NAPI统一管理这个问题：网卡驱动收到中断不需要自行决定如何收发，而是调用
napi_schedule...系列函数，比如napi_schedule_irqoff()，或者netif_reschedule系列
函数，比如netif_schedule_queue()，分别激活napi本身的接受或者发送调度。

这本质上分别激活了NET_RX_SOFTIRQ和NET_TX_SOFTIRQ两个softirq，然后在其中按一定
的配置平衡每次调度的数据的数量（通过/proc/sys/net/core/netdev_budget设置），用
napi->poll函数按指定的Budget进行调度。

NAPI对网卡驱动的接口大致如下：

* 保证net_device中有发送回调：::

        net_device->netdev_ops->ndo_start_xmit()

* 通过如下接口为每个通道（队列）建立一个调度上下文：::

        netif_napi_add();
        netif_napi_del();

  napi中需要提供一个poll函数，负责根据给定的Budget收报文。

* 用如下接口使能或者关闭收发功能：::

        napi_enable();
        napi_disable();

* poll函数负责从通道接收指定的Budget个数的报文，其中可以使用如下API：::

        napi_alloc_skb();       // 分配napi感知skb（可以cache化）
        napi_complete();        // 收到足够的报文，或者没有报文可接受时调用
        napi_complete_done();   // 报告完成了多少budget的版本，建议用这个版本
        napi_reschedule();      // 这是napi_schedule系列函数的poll内部使用版
                                // 在napi_complete...系列函数后请求再次调度用

* 在中断中用如下函数激活NAPI调度：::

        napi_schedule_irqoff(); // 用于硬中断已经关闭的情形
        napi_schedule();        // 用于硬中断未关闭的情形

  这组函数可以被拆成两步使用，本文忽略这种用法。

简单总结：驱动通过中断激活Softirq中的调度程序，Softirq关掉中断，按Budget统一调
度所有本CPU上的NAPI驱动进行polling，从而平衡IO和业务之间的计算压力。

todo：这个流程需要double check一次。


Ethtool接口
````````````
Ethtool是一个用户态的命令接口，用于设置Ethernet网卡的行为，比如读写EPROM，开启
关闭GRO等。在Linux内核中通过socket文件的ioctl()接口调用设备驱动的对应回调。

HNS3在Linux 5.5主线中支持的功能包括：

.. code-block:: c

   // drivers/net/ethernet/hisilicon/hns3/hns3_ethtool.c代码片段
   static const struct ethtool_ops hns3_ethtool_ops = {
	.self_test = hns3_self_test,
	.get_drvinfo = hns3_get_drvinfo,
	.get_link = hns3_get_link,
	.get_ringparam = hns3_get_ringparam,
	.set_ringparam = hns3_set_ringparam,
	.get_pauseparam = hns3_get_pauseparam,
	.set_pauseparam = hns3_set_pauseparam,
	.get_strings = hns3_get_strings,
	.get_ethtool_stats = hns3_get_stats,
	.get_sset_count = hns3_get_sset_count,
	.get_rxnfc = hns3_get_rxnfc,
	.set_rxnfc = hns3_set_rxnfc,
	.get_rxfh_key_size = hns3_get_rss_key_size,
	.get_rxfh_indir_size = hns3_get_rss_indir_size,
	.get_rxfh = hns3_get_rss,
	.set_rxfh = hns3_set_rss,
	.get_link_ksettings = hns3_get_link_ksettings,
	.set_link_ksettings = hns3_set_link_ksettings,
	.nway_reset = hns3_nway_reset,
	.get_channels = hns3_get_channels,
	.set_channels = hns3_set_channels,
	.get_coalesce = hns3_get_coalesce,
	.set_coalesce = hns3_set_coalesce,
	.get_regs_len = hns3_get_regs_len,
	.get_regs = hns3_get_regs,
	.set_phys_id = hns3_set_phys_id,
	.get_msglevel = hns3_get_msglevel,
	.set_msglevel = hns3_set_msglevel,
	.get_fecparam = hns3_get_fecparam,
	.set_fecparam = hns3_set_fecparam,
   };

这里不打算翻译Ethtool的用户手册，但我们在介绍HNS3的时候会再来看看部分典型功能
的工作原理。

DPDK
-----
todo

HNS3的设计
----------

HNS3是一个封装成PCIE接口的总线直连设备。这一节我们看看HNS3怎么为Linux内核提供
功能的。

HNS3的驱动子Linux内核主线的如下位置：::

        drivers/net/ethernet/hisilicon/hns3/*

这其中包含基于SR-IOV的PF和RF的不同设备发现，以及统一的网络驱动，要理解里面的代
码关系，我们可以先理解下面这个UML对象关系图：

        .. figure:: hns3_object_diagram.svg

.. CGE支持如下外链（全部全双工）：100G-Base-R, HIGig100, HiGig106, TransCode
   所有链路支持10x10和4x25两种模式，

配置完成后，BIOS，比如UEFI，会检测到不同的设备配置，虚拟PCIE总线枚举过程就会发
现他们，匹配上的PCIE设备注册为一个hnae3_dev，根据发现的设备是vf还是pf，选择不
同的硬件算法hnae3_algo，然后选择系统现在支持什么Client，注册为net_device或者
ib_device，或者同时注册为两者。简单说，硬件上只有一个PCIE设备，可以通过SR-IOV
接口创建更多的虚拟设备（Linux通过sysfs提供这个接口），每个设备可以有一个或者两
个Client，注册为net_device或者ib_device，这样就和Linux的网络或者InfiniBand子系
统结合起来了。

todo：需要一个两个100G全速的时候，全系统CPU占用率的数据。


多队列设计
````````````
HNS3通过物理层进行数模转化，把数据发送到网络上或者从网络中接收数据。

我们单独考察发送过程。前面提到，软件协议栈中的数据是内存中的sk_buff，所以最优
的通讯方式是CPU直接通知HNS3这个sk_buff的位置，让HNS3自己去读这个数据。

但通知sk_buff的位置，长度，分段等信息，需要把很多数据传递到硬件上。这个数据我
们称为Buffer Descriptor，简称BD。把BD告知硬件可以有两种方法：

1. 把请求直接发到设备上，这可以通过写MMIO空间实现。这个方法的缺点是需要在设备
   上留足够的资源，也需要有复杂的硬件设计去保证设备和CPU之间同步。

2. 把BD放在内存中，MMIO只起一个通知作用，两者通过这个共享内存来实现通讯。

HNS3使用的是后一种设计，但未来也有可能会发展为第一种设计。HNS3的BD在内存中组成
两个循环队列，一个用于发送，一个用于接收，这样就构成一对QP（Queue Pair），这称
为一个队列。

队列意味着一个软硬件通讯的上下文。它的通讯效率的上限是内存，CPU和HNS3硬件处理
线程的效率。只要能提升这三者的效率，通讯的效率就可以上升（无论是时延还是吞吐率
）。但如果这三者有不足，解决方案是提供多个队列，这可以提升效率。比如CPU处理效
率不足，大部分CPU时间都消耗在读写BD上了，我们创建2对QP，分别用两个CPU去读写不
同的QP，CPU上的压力就分解到两个不同的队列上了。

HNS3上有1024个硬件队列的资源，可以指配给不同的VF或者PF，这样这个对应的网卡上就
可以从多个队列同时收发报文。把每个队列对应的中断（如何确认）绑定到不同的CPU上
，就可以把CPU的处理压力分解开。

在作者写作本章的时候，鲲鹏920并没有提供特别好的手段匹配不同的队列，用户在Linux
中可以这样考虑找到对应队列的中断号：

* 从/proc/interrupts中找到<name>-TxRx-<nn>这样名字的中断。

* <name>对应一张网卡，而nn是它上面中断的id，发送和接收队列各占一个中断。所以
  0,1对应第一个队列，2, 3对应第二个队列，……如此类推。用这种方法可以匹配到这个
  网卡的每个队列的中断号。

如果把NUMA这个要素考虑在内，我们最好把中断绑定到靠近HNS3硬件的CPU上，这样CPU和
HNS3访问接近距离的NUMA内存。sk_buff分配的时候会优先在当前的NUMA节点上分配，所
以，对于发送，应用所在的节点影响sk_buff的NUMA位置，对于接收，接收中断把数据取
走后重新分配的时候用的是自己所在NUMA节点去分配，所以接收中断的执行位置决定了这
个sk_buff所在的NUMA节点。

这个结构，还有这样一些优化点：

* 从内存中读写数据还是慢，如果能直接从Cache中读写数据，这个速度会更快。

* 每次为了让设备看到，都需要做DMA操作，更新IOMMU中的页表和TLB，数据发送完了，
  还需要取消，避免非法的硬件对内核进行攻击。如果改用WarpDrive方案，这个DMA操作
  只会发生一次，这个性能就可以进一步提高。

这些优化，在后面的HNS软硬件升级中，也可能会逐步放到解决方案中。


分段下沉（Segmentation Offloading）
````````````````````````````````````

分段下沉是一个优化网卡和CPU接口的设计。数据报文在网络上传输，需要经过网桥，路
由器等设备的转发，报文的大小总是有限制的。这个限制称为这个网络的MTU，Maximum
Transfer Unit。传统Ethnernet MTU的默认大小是1500字节。这个大小针对不同的网络是
可以调整的，但考虑到互联网的复杂性和通用性，调整这个大小并不容易。

网络协议栈对报文进行切割，要以MTU为限度。但这个切割增加了CPU和网卡通讯的成本，
如果我们可以延迟这个切割，在网卡对外的接口上再完成这样的切割，就能提高CPU和网
卡间的通讯效率。这样的技术，就称为分段下沉。这是一系列技术的统称，比如：

TSO
        TCP Segmentation Offload。基于TCP的分段下沉，仅用于发送。

GSO
        Generic Segmentation Offload。Linux设计的包括TSO的通用分段下沉方案。

LRO
        Large Receive offload。基于TCP的接受方向的分段下沉。这个方案最大的问题
        在于合并接受的报文，会导致部分报文头部信息被合并和丢失，如果软件里有网
        桥一类的功能（如果有虚拟机，这个情况很常见），会引起各种问题。
        
GRO
        Generic Receive Offload。Linux的通用的接受方向方案。

除此以外，Linux还有其他细致的不同协议的下沉方案，这些可以从下面的内核文档中看
到部分介绍： ::

        Documentation/networking/segmentation-offloads.rst

HNS3实现了GSO和GRO的支持，其实现原理是……todo


Checksum下沉
``````````````
todo：介绍其他自动Checksum的算法是如何实现的，以其中一个作为例子，其他的带过即
可。

FD
````
todo

todo：ethtool的典型的，服务器使用中经常会碰到的功能的工作原理介绍。

.. vim: fo+=mM tw=78
