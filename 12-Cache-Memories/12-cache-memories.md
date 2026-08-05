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
#### Another Example
![[Pasted image 20260804152600.png|473]]
![[Pasted image 20260804152622.png|326]]
注意，取的地址一定是B-对齐的

- 当我们读**地址0**的字时：
由于缓存不命中，所以我们直接按B位对齐（从最开始每次走B字节，选取包含某个范围的B）
![[Pasted image 20260804152802.png|367]]
- 当我们读**地址1**的字时：
简单一点说，地址1的数据，即`m[1]`这包含在缓存里——命中。
复杂一点说，由于索引位一样且标记位一样，并且有效位为1——命中
**这里可以看出这种存储方式的有效性**——B的大小和索引位是匹配的，也就是说：一个B中能包含多个地址的数据（多个地址的数据在同一个B中），标记位一样的地址在同一个B中读，而标记位不同的地址不可能在同一个B中——这是被设计的）
- 当我们读**地址13**的字时：
![[Pasted image 20260804153501.png|358]]
注意这里的**B-对齐**
### 抖动
该程序几乎无法使用缓存，但原因并不是因为其空间局部性差
![[Pasted image 20260804161446.png|317]]
假设一个块是16字节，`x`被夹在到从的地址0开始的32字节连续内存中，而`y`紧跟其后
![[Pasted image 20260804161558.png|623]]
我们是先取x再取y，但是对于同下标的`x`,`y`，缓存每次读取都会被覆盖一次，每次都会**冲突不命中**——这种现象称之为**抖动**
解决的方法很简单，只需要在x后放B字节的padding，这样就能使得`x`和`y`完美错开（`y`紧随`x`后）
![[Pasted image 20260804161916.png]]
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
**write-hit**：使用缓存时，**CPU直接更改缓存**，下游**没有更改**，我们需要把这些更改传下去
两种方法：
- Write-through：立刻写入内存
- Write-back：等到某个块**将被覆盖时**才写入内存，这里我们需要一个dirty bit来指示这块数据是否已被写入。当缓存想要覆盖某个特定的行时，它会检查那个dirty bit，**如果其被设置**，那么就将数据写入磁盘。

**write-miss**：需要写入的内存不包含在缓存中
两种方法：
- Write-allocate：直接写入缓存
- No-write-allocate：直接修改主存数据
## Intel Core i7 Cache Hierarchy
![[Pasted image 20260725093856.png]]
此处的每一个Core都是并行，有自己的指令流
d/i-cache分别是data和instruction cache
## Cache Performance Metrics
- Miss Rate：缓存未命中的比例。对于L1约为3-10%，对L2则小于1%
- Hit Time：缓存命中时读取缓存所需要的时钟周期，包括决定某line是否在cache中的时间。对于L1约为4，对于L2约为10
- Miss Penalty：由于miss产生的额外时间花费，比如要算上查找缓存的时间。对于主存约为50-200

事实上，99%的命中率比97%的命中率要**好两倍**（miss penalty巨大）
## Writing Cache Friendly Code
查看最常被调用的函数，分析其中的循环（如果有）的循环体，我们尝试：
1. 最小化内层循环中的未命中，比如重复引用变量
> [!NOTE] 时间局部性
> 好的时间局部性：
> ```C
> use(a[0]);
> use(a[0]);
> use(a[0]);
> ```
> 
> 差的时间局部性：
> ```C
> use(a[0]);
> for (int i = 0; i < 100000000; i++) 
> 	use(large_array[i]);
> use(a[0]);
> ```
> 我们**并非是只看一个变量是否被重复使用**，毕竟只要是循环中的局部变量一定是被重复使用的。但是显然第二个代码中，缓存中的`a[0]`极其有可能在下面处理大量数据以后**被覆盖**

2. 步长为1的引用，因为Cache的块会把所需要地址周围的很多数据都放进去

这是无法列举完的，所以**理解缓存的工作方式**对优化代码有非常好的帮助
# Performance impact of caches
缓存对于代码性能的影响
## The Memory Mountain
- **Read throughput** (read bandwidth)：每秒从内存读取的字节数 (MB/s)
- **Memory Mountain**：将 **read throughput** 绘制为该循环中**空间和时间局部性的函数**

![[Pasted image 20260725110246.png]]
注意黄箭头线这里，我们会发现有一部分的平坦区域，这是由于步长过大（大于块长），每次都会命中不同的块，因此**丧失了空间局部性**
Size指的是**工作集**大小，也就是反复访问的那一整片数据区域的大小。只有小的工作集才能进比较高级的缓存中
红色箭头指向的Explode，是由于一种预测行为——缓存系统**识别到了这种步长为1的引用模式**，然后积极地抓取一堆块加载到缓存
## Rearranging loops
提高空间局部性
### Matrix Multiplication Example
![[Pasted image 20260725113121.png|299]]
#### Miss rate
假设：
- Matrix dimension = N（N非常大）
- Cache不足以存放乘起来的行

![[Pasted image 20260725113515.png|484]]
##### ijk
假设一个块能放四个整数
![[Pasted image 20260725113608.png|382]]
`a`是行访问，从第一次开始，每次放四个数到缓存，也就是第一次miss了以后，后面三次hit，然后不断循环。所以miss rate = 0.25
`b`是列访问，显然miss rate = 1.0
`c`不在内层循环里（没有k），不考虑
##### kij
![[Pasted image 20260725114550.png|373]]
对`c`和`b`都是行访问，所以miss rate都为0.25
##### jki
![[Pasted image 20260725114841.png|369]]
`a`和`c`的miss rate = 1.0
#### Summary
显然是第三种方法最区，我们完全不考虑。
但是仔细看看第一种和第二种，我们发现**每次循环**虽然前者的miss rate高于后者，但是后者比前者多一个store操作`+=`
写入操作比读取操作有更多的灵活性，所以其**耗费的时间也较少**（写入可以等待，因为他并不影响具体的操作，但是在获得数据之前，**CPU什么都做不了**）
![[Pasted image 20260725141151.png]]
## Using blocking
提高时间局部性——blocking（分块）
### Matrix Multiplication Example
![[Pasted image 20260725150331.png|542]]
#### Cache miss
![[Pasted image 20260725151616.png|478]]
对于一次迭代，第一个矩阵是$n/8$的miss，第二个矩阵是$n$的miss，加一起是$9n/8$的miss
由于一共要运行$n^2$次这样的迭代，所以总的miss是$9n^3/8$
### Blocked Matrix Multiplication
![[Pasted image 20260725152157.png|617]]
![[Pasted image 20260725152302.png|479]]
#### Cache miss
对于一次迭代，对于每一个矩阵，每一行都有$B/8$的miss，一共有$B$行，所以总共有$B^2/8$的miss。由于有两个矩阵，且一行有$n/B$个小矩阵。所以总共有$nB/4$
易知总miss为$n^3/(4B)$
#### Summary
![[Pasted image 20260725154610.png|167]]
不过我们不能让$B$过大，设缓存的大小为$C$，那么$3\times B^2<C$
# Summary
Cache由硬件管理，不过通过对Cache原理的了解，我们可以写出更多利用Cache的代码，从而提高效率