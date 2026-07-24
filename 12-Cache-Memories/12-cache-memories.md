# Cache memory organization and operation
![[Pasted image 20260724165327.png|490]]
## Cache Memories
![[Pasted image 20260724165652.png|473]]
**高速缓存存储器(cache memory)** 包含在CPU芯片里，完全由硬件管理
- 使用SRAM存储器实现
- CPU最先在此寻找数据
## General Cache Organization (S, E, B)
既然高速缓存存储器由硬件管理，那么硬件必须知道如何查找缓存的块，并且确定是否要包含特定的块
![[Pasted image 20260724170120.png|613]]

valid bit：指示这个数据块是有意义的（随机的垃圾值不是有意义的）
tag：帮助寻找块
注意：**v 和 tag 是单独的元数据，不包含在 B 中**
## Cache Read
当程序执行一条需要访问内存中某个字的数据指令时，CPU 会把这个字的内存地址**发送给缓存，并要求缓存返回该地址的字**（具体怎么返回由缓存负责）.
字的地址划分为三部分（这些区域由缓存的组织方式决定）
- 低位的b位决定了该字在块中开始的**偏移量**
- 接下来的s位被视为无符号整数，提供了**组索引**
- 最后的t位用于**帮助搜索**

![[Pasted image 20260724175519.png|575]]
注意，这里**并非是CPU知道这个地址**（即图中的"Address of word"），CPU给缓存的**就是内存地址**，只是因为**缓存在存数据的时候就把内存地址分成上面对应的部分来存储的**
### Example：Direct Mapped Cache (E=1)
![[Pasted image 20260724180753.png]]
![[Pasted image 20260724180817.png|547]]
Block 里存的是：**主存中一段连续地址的原始字节数据**。每次读取的数据是**从按B字节对齐的内存开始**(block start=$⌊A/B​⌋B$)，往后读B位。此外，**读取的这段数据从Block的0-offset开始存，而不是Address of word的block offset开始存**
这也就意味着，其并**不一定就是那个内存里的数据**，因为他可能包含前后的一些数据
#### Direct-Mapped Cache Simulation
![[Pasted image 20260724193816.png|486]]
初始的Cache：
![[Pasted image 20260724194007.png|241]]

过程：
先看第一个地址：由于`v=0`所以miss。然后把这个数据放入Cache——`Tag=0`,`Block=M[0-1]`（这个表示内存中从偏移0开始到块结束包含的字节），同时`v=1`
再看第二个地址：由于`s=0`，`v=1`且`Tag=0`，所以hit
再看第三个地址：由于`s=3`而`v=0`，所以miss。然后`v=1`,`Tag=0`,`Block=M[6-7]`
......

最后的结果：
![[Pasted image 20260724200832.png|304]]
在此处可以仔细思考一下为什么第二个地址hit了？
### Example：E-way Set Associative Cache (E=2)
![[Pasted image 20260724200941.png]]
只看Set1：
![[Pasted image 20260724201121.png|519]]
![[Pasted image 20260724202032.png|516]]
#### 2-Way Set Associative Cache Simulation
![[Pasted image 20260724202237.png|570]]
初始的Cache：
![[Pasted image 20260724202210.png|282]]
> [!NOTE] 注意Q&A
> Q：**到底读多少字节？**
> A：`mov`指令——`movb, movw, movl, movq`，所以Cache知道自己要读多少的字节
> 
> Q：**应该用哪个行？**
> A：有各种各样的算法，一种简单的算法就是覆盖长时间未被访问那一行（课上不考虑这一点，我们使用随机放置）
> 
> Q：**B是否越大越好呢？**
> A：并非，B越大，从内存中读取的数据越多，浪费就越多；但是也不能太小，不然就需要跨块
## Cache Write



