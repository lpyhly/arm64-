# execl测试项解析

# 热点

## 4k页表

### 单核

#### 1630V100样片-160c

测试环境： 123#  openEuler 22.03， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）



### 多核

#### 1630V100样片-160c

测试环境： 123#  openEuler 22.03， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

分数： 6636分

特征：

- 带宽很低，只有几GB；
- 存在很高的FE bound，由icache miss带来的，需要分析。
- 

```shell
  35.33%  [kernel]               [k] osq_lock
   7.80%  [kernel]               [k] filemap_map_pages
   5.70%  [kernel]               [k] release_pages
   4.11%  [kernel]               [k] page_remove_file_rmap
   4.01%  [kernel]               [k] ptep_clear_flush
   3.56%  [kernel]               [k] rwsem_spin_on_owner
   2.65%  [kernel]               [k] vma_interval_tree_insert
   1.61%  [kernel]               [k] zap_pte_range
   1.25%  [kernel]               [k] change_protection_range
   1.10%  [kernel]               [k] up_write
   1.02%  [kernel]               [k] tlb_flush_mmu
   0.97%  [kernel]               [k] rwsem_wake.constprop.0.isra.0
   0.95%  [kernel]               [k] __handle_mm_fault
   0.85%  [kernel]               [k] vma_interval_tree_remove
```

``` shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
        Total Execution Instruction:                    92099746411
        Total Execution Cycles:                        251524123071
        Instructions Per Cycle:                               0.366

        Front-End Bound:                                    44.909%
                Front-End Latency:                          43.587%
                        Idle_by_itlb_miss:                   1.190%
                        Idle_by_icache_miss:                15.294%
                        BP_Misp_Flush:                       0.850%
                        OoO_Flush:                           0.162%
                        SP_Flush:                            0.801%
                Front End Bound Bandwidth:                   1.323%

        Bad Speculation:                                     2.168%
                Branch Mispredicts:                          1.961%
                        Indirect Branch:                     0.825%
                        Push Branch:                         0.636%
                        Pop Branch:                          0.004%
                        Other Branch:                      100.000%
                Machine Clears:                              0.207%
                        Nuke Flush:                         14.280%
                        Other Flush:                        85.720%

        Retiring:                                            6.103%

        Back-End Bound:                                     46.820%
                Resource Bound:                             15.806%
                        Rob_stall:                          11.428%
                        Ptag_stall:                          0.166%
                        MapQ_stall:                         86.630%
                        PCBuf_stall:                         1.776%
                        Other_stall:                         0.000%
                Core Bound:                                 38.286%
                        FDIV_Stall:                          0.000%
                        DIV_Stall:                           0.005%
                        FSU_stall:                           0.000%
                        Exe Ports Util:                     38.281%
                                0_ports_serialize:           7.763%
                                0_ports_non_serialize:      78.044%
                                1_ports:                     2.737%
                                2_ports:                     2.889%
                                3_ports:                     2.600%
                                4_ports:                     2.215%
                                5_ports:                     1.630%
                                6_ports:                     2.228%
                Memory Bound:                               43.950%
                        L1 Bound:                           25.222%
                                DTLB:                        0.317%
                                Misalign:                    0.029%
                                Resource_Full:               0.002%
                                Instruction_Type:            0.212%
                                Forward_hazard:              3.168%
                                Structure_hazard:            1.332%
                                Pipeline:                    0.164%
                        L2 Bound:                            1.056%
                                buffer pending:              0.000%
                                snoop pending:               0.094%
                                Arb idle:                    0.924%
                                Pipeline:                    0.038%
                        L3 Bound:                           17.672%
                        Mem Bound:                           0.000%
                                Local Dram:                  0.000%
                                Remote Dram:                 0.000%
                                Remote Cache:                0.000%
                        Store Bound:                         0.000%
                                SCA:                         0.781%
                                Head:                       12.250%
                                Order:                      64.212%
                                Other:                       0.046%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

```

``` python
******************DDR*******************************
TIME    SCCL    DDR_READ        DDR_WRITE       MATA_TOTAL      CROSS_CHIP      CROSS_DIE       INNER_DIE
4       sccl1   0.434 GB/s      1.158 GB/s      1.253 GB/s      0.190 GB/s      0.306 GB/s      0.757 GB/s
4       sccl11  0.332 GB/s      0.847 GB/s      0.926 GB/s      0.119 GB/s      0.195 GB/s      0.612 GB/s
4       sccl9   2.163 GB/s      3.519 GB/s      4.113 GB/s      1.077 GB/s      1.289 GB/s      1.746 GB/s
4       sccl3   0.454 GB/s      1.038 GB/s      1.190 GB/s      0.194 GB/s      0.259 GB/s      0.736 GB/s
********************L3C*****************************
TIME    SCCL    CPIPE_READ      CPIPE_WRI       CPIPE_HIT_R     CPIPE_hit_W     SPIPE_READ      SPIPE_WRI       SPIPE_hit_R     SPIPE_hit_W     L3_HIT_ALL      RET_TO_CPU_DATA
4       sccl1   3.174 GB/s      0.000 GB/s      0.176 GB/s      0.000 GB/s      3.186 GB/s      0.000 GB/s      1.221 GB/s      0.000 GB/s      2.546 GB/s      3.073 GB/s
4       sccl11  3.131 GB/s      0.000 GB/s      0.254 GB/s      0.000 GB/s      3.141 GB/s      0.000 GB/s      1.179 GB/s      0.000 GB/s      2.414 GB/s      3.033 GB/s
4       sccl9   3.176 GB/s      0.000 GB/s      0.259 GB/s      0.000 GB/s      3.173 GB/s      0.000 GB/s      1.212 GB/s      0.000 GB/s      2.464 GB/s      3.082 GB/s
4       sccl3   3.215 GB/s      0.000 GB/s      0.177 GB/s      0.000 GB/s      3.200 GB/s      0.000 GB/s      1.214 GB/s      0.000 GB/s      2.582 GB/s      3.118 GB/s

```





##### TopDown





## 64k页表

### 单核

### 多核

#### 1630V100样片-160c

测试环境： 123#  openEuler 22.03， pagesize 4k，开预取，关SMT，160进程on 2P 160c，2.9GHz CPU频率 + 16 * 32GB DDR（5200，2Rx* RDIMM）

##### 热点

重点函数在osq_lock & spinlock&rwsem上。

```shell
    25.16%  [kernel]       [k] osq_lock                                         
    11.09%  [kernel]       [k] queued_spin_lock_slowpath                        
    10.27%  [kernel]       [k] arch_cpu_idle                                    
     3.54%  [kernel]       [k] rwsem_spin_on_owner                              
     2.36%  [kernel]       [k] vma_interval_tree_insert                         
     1.97%  [kernel]       [k] clear_page                                       
     1.89%  [kernel]       [k] copy_page                                        
     1.57%  [kernel]       [k] __d_lookup_rcu                                   
     1.38%  [kernel]       [k] page_cache_get_speculative                       
     1.26%  [kernel]       [k] up_write                                         
     1.06%  [kernel]       [k] try_to_wake_up                                   
     0.97%  [kernel]       [k] rwsem_wake.isra.0                                
     0.91%  [kernel]       [k] sched_exec                                       
     0.87%  [kernel]       [k] vma_interval_tree_remove                         
     0.86%  [kernel]       [k] lockref_get_not_dead                             
     0.83%  [kernel]       [k] finish_task_switch                               
     0.82%  [kernel]       [k] handle_mm_fault                                  
     0.76%  [kernel]       [k] ptep_clear_flush                                 
     0.75%  [kernel]       [k] down_write                                       
     0.75%  [kernel]       [k] __arch_clear_user                                
```

##### Topdown

发现前端bound很严重，尤其是icache miss，为什么？

```shell
+++++++++++++++++++++Top-down Microarchitecture Analysis Summary+++++++++++++++++
	Total Execution Instruction:                    95268545509
	Total Execution Cycles:                        253240160298
	Instructions Per Cycle:                               0.376

	Front-End Bound:                                    43.843%
		Front-End Latency:                          42.507%
			Idle_by_itlb_miss:                   0.822%
			Idle_by_icache_miss:                16.148%
			BP_Misp_Flush:                       0.873%
			OoO_Flush:                           0.254%
			SP_Flush:                            0.809%
		Front End Bound Bandwidth:                   1.337%

	Bad Speculation:                                     2.328%
		Branch Mispredicts:                          2.004%
			Indirect Branch:                     1.290%
			Push Branch:                         0.724%
			Pop Branch:                          0.007%
			Other Branch:                      100.000%
		Machine Clears:                              0.324%
			Nuke Flush:                          9.940%
			Other Flush:                        90.060%

	Retiring:                                            6.270%

	Back-End Bound:                                     47.558%
		Resource Bound:                             15.462%
			Rob_stall:                           7.707%
			Ptag_stall:                          0.194%
			MapQ_stall:                         90.232%
			PCBuf_stall:                         1.866%
			Other_stall:                         0.000%
		Core Bound:                                 38.083%
			FDIV_Stall:                          0.009%
			DIV_Stall:                           0.006%
			FSU_stall:                           0.000%
			Exe Ports Util:                     38.077%
				0_ports_serialize:          16.334%
				0_ports_non_serialize:      69.259%
				1_ports:                     2.947%
				2_ports:                     2.782%
				3_ports:                     2.251%
				4_ports:                     2.390%
				5_ports:                     1.912%
				6_ports:                     1.726%
		Memory Bound:                               44.197%
			L1 Bound:                           19.239%
				DTLB:                        0.394%
				Misalign:                    0.040%
				Resource_Full:               0.018%
				Instruction_Type:            0.306%
				Forward_hazard:              2.604%
				Structure_hazard:            0.966%
				Pipeline:                    0.130%
			L2 Bound:                            1.306%
				buffer pending:              0.000%
				snoop pending:               0.086%
				Arb idle:                    1.061%
				Pipeline:                    0.158%
			L3 Bound:                           22.648%
			Mem Bound:                           0.000%
				Local Dram:                  0.000%
				Remote Dram:                 0.000%
				Remote Cache:                0.000%
			Store Bound:                         1.004%
				SCA:                         3.041%
				Head:                        3.397%
				Order:                      70.665%
				Other:                       0.105%
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

