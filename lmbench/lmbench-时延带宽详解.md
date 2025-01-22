#lmbench时延带宽

## wr

### 测试项原理

每次写4Byte，每隔16Byte写一次。汇编代码包含str/stur,前者用于偏移为正数，后者用于偏移为负数。

软件上统计带宽时，不会按照总共写了多少byte，而是按照总的地址空间来计算。比如总地址空间为64Byte，则需要写4次，共写4*4=16Byte的数，但是这里计算带宽是按照4次str总共写了64Byte地址的原则。

非full cacheline写，所以不会触发streaming。

16Byte写一次，一条cacheline需要4个str去写4次。那么，SCB会先从DC中refill一条完整cacheline，然后将STQ中该cacheline包含的4个entry都merge进来，最后再写回到DC去。

### L1 cache理论值

​	从LS IssueQ将sta issue到stq的速率为 2个/cycle， DAGB到stq的写口为2个/cycle， stq->scb的写口为2个/cycle。这三个因素共同决定了，L1$ hit的场景下，理论带宽为 16Byte * 2/cycle.

### L2 cache理论值

​	需要考虑SCB的entry size和大小。

- 在910时，SCBD为16个，每个entry是16Byte，而wr的每个操作都会占一个entry，因此16个操作（8 cycle）可以把SCB占满。 而L2时延为13 cycle，因此瓶颈不在stq，而是在SCB了。
  - 理论带宽：  (outstanding / latency)   16 ops * 8Byte / 13 cycle 
- 在onepro时，每个entry为32Byte，因此2个操作共用一个entry，需要32个操作才会把SCB占满。由此，scb的最大带宽是高于stq的，瓶颈依然在stq，那么理论带宽与L1 cache一致。

### L3 cache理论值

看不懂，为什么会stq+scbd entry（28 + 16）？？？





## fwr

### 测试项原理

每次写4Byte，连续写。 汇编代码包含str/stur,前者用于偏移为正数，后者用于偏移为负数。

因为是full cacheline的写，因此会触发streaming store，直接发WriteUnique下去。

