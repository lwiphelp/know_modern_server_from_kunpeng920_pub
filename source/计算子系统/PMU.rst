.. Copyright by Kenneth Lee. 2020. All Right Reserved.

PMU
===

PMU是Performance Monitor Unit的缩写。它是一个事件统计器，里面包含了一组计数器
，计算和总线子系统的在处理过程中发生的事件都可以记录到PMU中。PMU作为一个设备，
也可以在计数达到一个特定门限的时候对CPU产生中断。

在实现的时候，PMU实际上是很多个独立的计数单元的综合体，示意如下图：

        .. figure:: kp920_pmu_arch.svg

可以看到，这是一个离散的组织，我们可以从Linux内核代码中查看这个硬件实现。首先
核内的PMU符合ARM的标准，使用的是ARM的驱动，在这里：::

        arch/arm64/kernel/perf_event.c

核外的是鲲鹏920专有驱动，在如下位置：::

        >ls drivers/perf/hisilicon
        hisi_uncore_ddrc_pmu.c  hisi_uncore_hha_pmu.c  hisi_uncore_l3c_pmu.c
        hisi_uncore_pmu.c  hisi_uncore_pmu.h  Makefile

这里有三个独立的驱动，每个我们都可以找到它们对应的ACPI设备名：

* DDRC上的：HISI0233
* HHA上的：HISI0243
* L3C上的：HISI0213

然后我们到运行系统（基于Linux的话）的sys/bus/platform/devices目录中就可以找到
每个具体实体的实例了。

基于这样一个统计系统，我们可以有两种方式来使用这个装置。一种是统计，比如你调用
一个函数，或者运行一个程序，我们几个在运行前记录一个统计值，结束的时候记录另一
个值，两者相减，就可以知道这个函数或者程序触发了多少这个事件了。

当然，如果中间发生了调度，还需要有办法减去中间的过程，但这并不难做到。下面是在
在鲲鹏920的Linux环境中，运行perf stat命令收集tree命令运行的统计结果示例：::

 Performance counter stats for 'tree':

              6.11 msec task-clock                #    0.773 CPUs utilized          
                 4      context-switches          #    0.655 K/sec                  
                 0      cpu-migrations            #    0.000 K/sec                  
               118      page-faults               #    0.019 M/sec                  
         4,518,608      cycles                    #    0.740 GHz                    
         4,122,933      instructions              #    0.91  insn per cycle         
   <not supported>      branches                                                    
            35,192      branch-misses                                               

       0.007906460 seconds time elapsed

       0.003721000 seconds user
       0.003721000 seconds sys

对应统计，另一种使用方式是采样。也就是说，我们不计算一个统计结果。我们反过来，
每次计数器统计满了一定的数量，基于中断报告，我们可以检查一下这个事件是哪个地方
触发的，这样概率上我们能发现触发这个事情的热点。

这个原理可以示意如下：

        .. figure:: perf_sampling.svg

很明显，采样方式并不准确，我们首先需要它的样本足够大才有参考价值。另一方面，它
还受多个其他要素的影响：

1. PMU的中断是异步中断，当CPU收到这个事件的时候，可能触发热点的指令已经结束了
   ，采样程序可能采样到后面的位置上。

2. 鲲鹏920是一个超流水线系统，流水线中包括多条指令，采样程序可能会随机采样到其
   中一条指令，而不是触发采样的那条指令。

3. PMU中断可能被关中断程序临时阻拦，采样报告中断可能需要等到中断在开中断的时候
   才报到CPU中，这个热点就会走到开中断程序。

这些问题都有技术可以解决，但对应也有成本，鲲鹏920上没有去解决它们。技术如此，
这个技术仍可以解决很多问题。下图展示了使用perf record命令统计lmbench stream测
试用例的时候的输出：::

        # Total Lost Samples: 0
        #
        # Samples: 41K of event 'cycles'
        # Event count (approx.): 26780433545
        #
        # Overhead  Command  Shared Object      Symbol                                     
        # ........  .......  .................  ...........................................
        #
            12.82%  stream   stream             [.] 0x000000000000187c
            11.22%  stream   stream             [.] 0x000000000000181c
             7.85%  stream   [kernel.kallsyms]  [k] ptep_clear_flush
             5.99%  stream   stream             [.] 0x00000000000017c4
             5.76%  stream   stream             [.] 0x000000000000176c
             4.84%  stream   [kernel.kallsyms]  [k] clear_page
             4.74%  stream   stream             [.] 0x0000000000001814
             4.53%  stream   [kernel.kallsyms]  [k] copy_page
             4.37%  stream   stream             [.] 0x0000000000001874
             3.33%  stream   stream             [.] 0x0000000000001768
             3.12%  stream   stream             [.] 0x00000000000017bc
             2.08%  stream   stream             [.] 0x0000000000001810
             1.88%  stream   stream             [.] 0x000000000000186c
             1.43%  stream   [kernel.kallsyms]  [k] get_page_from_freelist
             1.29%  stream   stream             [.] 0x0000000000001a08
             1.24%  stream   [kernel.kallsyms]  [k] el0_da
             1.18%  stream   [kernel.kallsyms]  [k] free_unref_page_list
             1.15%  stream   [kernel.kallsyms]  [k] release_pages
             0.90%  stream   [kernel.kallsyms]  [k] change_protection_range
             0.86%  stream   [kernel.kallsyms]  [k] free_unref_page
             0.80%  stream   stream             [.] 0x0000000000001a1c
             0.79%  stream   [kernel.kallsyms]  [k] zap_pte_range
             0.76%  stream   [kernel.kallsyms]  [k] pagevec_lru_move_fn
             0.67%  stream   stream             [.] 0x0000000000001a10
             0.64%  stream   [kernel.kallsyms]  [k] isolate_lru_page
             ...

测试时我们没有stream的符号，这里只显示了内核采样到的函数的名字。可以看到，在这
个内存访问的测试中，主要时间都消耗在stream的用户程序中。但内核的
ptep_clear_flush函数也占据7.85%的时间，说明缺页分配也占据了相当的CPU执行时间。

上面这个统计是用CPU cycle作为采样的，实际上，鲲鹏920上支持的PMU时间相当多。

在Linux中，通过perf命令可以列出鲲鹏920支持的所有硬件计数器：::

          branch-misses                                      [Hardware event]
          bus-cycles                                         [Hardware event]
          cache-misses                                       [Hardware event]
          cache-references                                   [Hardware event]
          cpu-cycles OR cycles                               [Hardware event]
          instructions                                       [Hardware event]
          stalled-cycles-backend OR idle-cycles-backend      [Hardware event]
          stalled-cycles-frontend OR idle-cycles-frontend    [Hardware event]
          alignment-faults                                   [Software event]
          bpf-output                                         [Software event]
          context-switches OR cs                             [Software event]
          cpu-clock                                          [Software event]
          cpu-migrations OR migrations                       [Software event]
          dummy                                              [Software event]
          emulation-faults                                   [Software event]
          major-faults                                       [Software event]
          minor-faults                                       [Software event]
          page-faults OR faults                              [Software event]
          task-clock                                         [Software event]
          duration_time                                      [Tool event]
          L1-dcache-load-misses                              [Hardware cache event]
          L1-dcache-loads                                    [Hardware cache event]
          L1-icache-load-misses                              [Hardware cache event]
          L1-icache-loads                                    [Hardware cache event]
          branch-load-misses                                 [Hardware cache event]
          branch-loads                                       [Hardware cache event]
          dTLB-load-misses                                   [Hardware cache event]
          dTLB-loads                                         [Hardware cache event]
          iTLB-load-misses                                   [Hardware cache event]
          iTLB-loads                                         [Hardware cache event]
          armv8_pmuv3_0/br_mis_pred/                         [Kernel PMU event]
          armv8_pmuv3_0/br_mis_pred_retired/                 [Kernel PMU event]
          armv8_pmuv3_0/br_pred/                             [Kernel PMU event]
          armv8_pmuv3_0/br_retired/                          [Kernel PMU event]
          armv8_pmuv3_0/br_return_retired/                   [Kernel PMU event]
          armv8_pmuv3_0/bus_access/                          [Kernel PMU event]
          armv8_pmuv3_0/bus_cycles/                          [Kernel PMU event]
          armv8_pmuv3_0/cid_write_retired/                   [Kernel PMU event]
          armv8_pmuv3_0/cpu_cycles/                          [Kernel PMU event]
          armv8_pmuv3_0/dtlb_walk/                           [Kernel PMU event]
          armv8_pmuv3_0/exc_return/                          [Kernel PMU event]
          armv8_pmuv3_0/exc_taken/                           [Kernel PMU event]
          armv8_pmuv3_0/inst_retired/                        [Kernel PMU event]
          armv8_pmuv3_0/inst_spec/                           [Kernel PMU event]
          armv8_pmuv3_0/itlb_walk/                           [Kernel PMU event]
          armv8_pmuv3_0/l1d_cache/                           [Kernel PMU event]
          armv8_pmuv3_0/l1d_cache_refill/                    [Kernel PMU event]
          armv8_pmuv3_0/l1d_cache_wb/                        [Kernel PMU event]
          armv8_pmuv3_0/l1d_tlb/                             [Kernel PMU event]
          armv8_pmuv3_0/l1d_tlb_refill/                      [Kernel PMU event]
          armv8_pmuv3_0/l1i_cache/                           [Kernel PMU event]
          armv8_pmuv3_0/l1i_cache_refill/                    [Kernel PMU event]
          armv8_pmuv3_0/l1i_tlb/                             [Kernel PMU event]
          armv8_pmuv3_0/l1i_tlb_refill/                      [Kernel PMU event]
          armv8_pmuv3_0/l2d_cache/                           [Kernel PMU event]
          armv8_pmuv3_0/l2d_cache_refill/                    [Kernel PMU event]
          armv8_pmuv3_0/l2d_cache_wb/                        [Kernel PMU event]
          armv8_pmuv3_0/l2d_tlb/                             [Kernel PMU event]
          armv8_pmuv3_0/l2d_tlb_refill/                      [Kernel PMU event]
          armv8_pmuv3_0/l2i_cache/                           [Kernel PMU event]
          armv8_pmuv3_0/l2i_cache_refill/                    [Kernel PMU event]
          armv8_pmuv3_0/l2i_tlb/                             [Kernel PMU event]
          armv8_pmuv3_0/l2i_tlb_refill/                      [Kernel PMU event]
          armv8_pmuv3_0/ll_cache/                            [Kernel PMU event]
          armv8_pmuv3_0/ll_cache_miss/                       [Kernel PMU event]
          armv8_pmuv3_0/ll_cache_miss_rd/                    [Kernel PMU event]
          armv8_pmuv3_0/ll_cache_rd/                         [Kernel PMU event]
          armv8_pmuv3_0/mem_access/                          [Kernel PMU event]
          armv8_pmuv3_0/memory_error/                        [Kernel PMU event]
          armv8_pmuv3_0/remote_access/                       [Kernel PMU event]
          armv8_pmuv3_0/remote_access_rd/                    [Kernel PMU event]
          armv8_pmuv3_0/sample_collision/                    [Kernel PMU event]
          armv8_pmuv3_0/sample_feed/                         [Kernel PMU event]
          armv8_pmuv3_0/sample_filtrate/                     [Kernel PMU event]
          armv8_pmuv3_0/sample_pop/                          [Kernel PMU event]
          armv8_pmuv3_0/stall_backend/                       [Kernel PMU event]
          armv8_pmuv3_0/stall_frontend/                      [Kernel PMU event]
          armv8_pmuv3_0/sw_incr/                             [Kernel PMU event]
          armv8_pmuv3_0/ttbr_write_retired/                  [Kernel PMU event]
          hisi_sccl1_ddrc0/act_cmd/                          [Kernel PMU event]
          hisi_sccl1_ddrc0/flux_rcmd/                        [Kernel PMU event]
          hisi_sccl1_ddrc0/flux_rd/                          [Kernel PMU event]
          hisi_sccl1_ddrc0/flux_wcmd/                        [Kernel PMU event]
          hisi_sccl1_ddrc0/flux_wr/                          [Kernel PMU event]
          hisi_sccl1_ddrc0/pre_cmd/                          [Kernel PMU event]
          hisi_sccl1_ddrc0/rnk_chg/                          [Kernel PMU event]
          hisi_sccl1_ddrc0/rw_chg/                           [Kernel PMU event]
          hisi_sccl1_ddrc1/act_cmd/                          [Kernel PMU event]
          ...
          exe_stall_cycle                                   
               [Cycles of that the number of issue ups are less than 4]
          fetch_bubble                                      
               [Instructions can receive, but not send]
          hit_on_prf                                        
               [Hit on prefetched data]
          if_is_stall                                       
               [Instruction fetch stall cycles]
          iq_is_empty                                       
               [Instruction queue is empty]
          l1d_cache_inval                                   
               [L1D cache invalidate]
          l1d_cache_rd                                      
               [L1D cache access, read]
          l1d_cache_refill_rd                               
               [L1D cache refill, read]
          l1d_cache_refill_wr                               
               [L1D cache refill, write]
          l1d_cache_wb_clean                                
               [L1D cache Write-Back, cleaning and coherency]
          l1d_cache_wb_victim                               
               [L1D cache Write-Back, victim]
          l1d_cache_wr                                      
               [L1D cache access, write]
          l1d_tlb_rd                                        
               [L1D tlb access, read]
          l1d_tlb_refill_rd                                 
               [L1D tlb refill, read]
          l1d_tlb_refill_wr                                 
               [L1D tlb refill, write]
          l1d_tlb_wr                                        
               [L1D tlb access, write]
          l1i_cache_prf                                     
               [L1I cache prefetch access count]
          l1i_cache_prf_refill                              
               [L1I cache miss due to prefetch access count]
          l2d_cache_inval                                   
               [L2D cache invalidate]
          l2d_cache_rd                                      
               [L2D cache access, read]
          l2d_cache_refill_rd                               
               [L2D cache refill, read]
          l2d_cache_refill_wr                               
               [L2D cache refill, write]
          l2d_cache_wb_clean                                
               [L2D cache Write-Back, cleaning and coherency]
          l2d_cache_wb_victim                               
               [L2D cache Write-Back, victim]
          l2d_cache_wr                                      
               [L2D cache access, write]
          mem_stall_anyload                                 
               [No any micro operation is issued and meanwhile any load operation is
                not resolved]
          mem_stall_l1miss                                  
               [No any micro operation is issued and meanwhile there is any load
                operation missing L1 cache and pending data refill]
          mem_stall_l2miss                                  
               [No any micro operation is issued and meanwhile there is any load
                operation missing both L1 and L2 cache and pending data refill from L3
                cache]
          prf_req                                           
               [Prefetch request from LSU]

          uncore ddrc:
          uncore_hisi_ddrc.act_cmd                          
               [DDRC active commands. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.flux_rcmd                        
               [DDRC read commands. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.flux_rd                          
               [DDRC total read operations. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.flux_wcmd                        
               [DDRC write commands. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.flux_wr                          
               [DDRC total write operations. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.pre_cmd                          
               [DDRC precharge commands. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.rnk_chg                          
               [DDRC rank commands. Unit: hisi_sccl,ddrc]
          uncore_hisi_ddrc.rw_chg                           
               [DDRC read and write changes. Unit: hisi_sccl,ddrc]

          uncore hha:
          uncore_hisi_hha.rd_ddr_128b                       
               [The number of read operations sent by HHA to DDRC which size is 128
                bytes. Unit: hisi_sccl,hha]
          uncore_hisi_hha.rd_ddr_64b                        
               [The number of read operations sent by HHA to DDRC which size is 64
                bytes. Unit: hisi_sccl,hha]
          uncore_hisi_hha.rx_ccix                           
               [Count of the number of operations that HHA has received from CCIX.
                Unit: hisi_sccl,hha]
          uncore_hisi_hha.rx_ops_num                        
               [The number of all operations received by the HHA. Unit: hisi_sccl,hha]
          uncore_hisi_hha.rx_outer                          
               [The number of all operations received by the HHA from another socket.
                Unit: hisi_sccl,hha]
          uncore_hisi_hha.rx_sccl                           
               [The number of all operations received by the HHA from another SCCL in
                this socket. Unit: hisi_sccl,hha]
          uncore_hisi_hha.spill_num                         
               [Count of the number of spill operations that the HHA has sent. Unit:
                hisi_sccl,hha]
          uncore_hisi_hha.spill_success                     
               [Count of the number of successful spill operations that the HHA has
                sent. Unit: hisi_sccl,hha]
          uncore_hisi_hha.wr_ddr_128b                       
               [The number of write operations sent by HHA to DDRC which size is 128
                bytes. Unit: hisi_sccl,hha]
          uncore_hisi_hha.wr_ddr_64b                        
               [The number of write operations sent by HHA to DDRC which size is 64
                bytes. Unit: hisi_sccl,hha]

          uncore l3c:
          uncore_hisi_l3c.back_invalid                      
               [Count of the number of L3C back invalid operations. Unit:
                hisi_sccl,l3c]
          uncore_hisi_l3c.prefetch_drop                     
               [Count of the number of prefetch drops from this L3C. Unit:
                hisi_sccl,l3c]
          uncore_hisi_l3c.rd_cpipe                          
               [Total read accesses. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.rd_hit_cpipe                      
               [Total read hits. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.rd_hit_spipe                      
               [Count of the number of read lines that hits in spipe of this L3C.
                Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.rd_spipe                          
               [Count of the number of read lines that come from this cluster of CPU
                core in spipe. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.retry_cpu                         
               [Count of the number of retry that L3C suppresses the CPU operations.
                Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.retry_ring                        
               [Count of the number of retry that L3C suppresses the ring operations.
                Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.victim_num                        
               [l3c precharge commands. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.wr_cpipe                          
               [Total write accesses. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.wr_hit_cpipe                      
               [Total write hits. Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.wr_hit_spipe                      
               [Count of the number of write lines that hits in spipe of this L3C.
                Unit: hisi_sccl,l3c]
          uncore_hisi_l3c.wr_spipe                          
               [Count of the number of write lines that come from this cluster of CPU
                core in spipe. Unit: hisi_sccl,l3c]

这个列表极长，这里删除大部分因为多个实体存在产生的重复。

Linux内核不但支持perf，也支持Oprofile，这些命令都可以利用PMU的能力。

.. vim: fo+=mM tw=78
