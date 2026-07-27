# Storage technologies and trends
系统提供了一个抽象，把内存结构抽象为一个大的线性数组
## Random-Access Memory (RAM)
- RAM被包装为芯片，多个RAM芯片组成**主存**
- RAM的基本存储单元称为**单元**(cell)，大小为1bit
- RAM分为两种类型：**S**RAM(Static)、**D**RAM(Dynamic)
![[Pasted image 20260722113843.png|577]]
refresh指以一定电压冲入电荷，如果DRAM没有refresh就会丢失电荷，导致丢失信息；EDC指Error Detaction&Correction，可以看出SRAM会比DRAM更可靠
### SRAM
（这一章书的翻译很不好）
SRAM将每个位储存在一个双稳态的存储器单元里。他可以**无限期保持**在两个不同的电压配置之一
![[Pasted image 20260726171424.png|512]]
**只要有电，SRAM就会永远保持他的值**，就算有干扰来扰乱电压，当其消失后，电路会迅速恢复稳定
### DRAM
DRAM将每个位存储为对一个电容的充电。由于漏电，DRAM单元在10~100ms内会**失去电荷**，内存系统必须**周期性读出、重写以刷新内存(refresh)**
### 传统的 DRAM
![[Pasted image 20260726173025.png|432]]
DRAM芯片被分成$d$个**超单元**(supercell)，每个超单元由$w$个**DRAM单元**(1bit)组成
信息通过被称为 **引脚(pin)** 的外部连接器进出芯片，每个引脚携带1 bit的信号。图中就是8个data引脚和2个addr引脚（携带两位的行**或**列超单元地址，也就是**分别传送**行和列的地址——RAS和CAS）

每个DRAM芯片都被连接到**内存控制器**，他可以一次传送w bit进或出DRAM芯片
例如，内存控制器会先发送**RAS**，然后DRAM芯片会直接**把一整行都放入缓冲区**，接着内存控制器发送**CAS**，DRAM芯片就会把放在缓冲区的对应列从data引脚发送到内存控制器。
![[Pasted image 20260726181225.png]]
### 内存模块
DRAM芯片封装在*内存模块*(memory module)中，其被插在主板的扩展槽上。
![[Pasted image 20260726201412.png|565]]
每个超单元存储主存的一个字节，而用相应超单元为$(i,j)$的8个超单元（图中的8个蓝色小方形）**表示主存中字节地址A处的64位字**。
要取出内存地址A处的一个字，**内存控制器**将其转换为**超单元地址**$(i,j)$，并将其发送到**内存模块**，然后内存模块将 $i$ 和 $j$ 广播到每一个DRAM。然后每个DRAM输出他对应地址的字节，模块中的电路会收集这些输出，把它们合并为64位字后返回给内存控制器
## Nonvolatile Memories
DRAM和SRAM都是易失性存储器，**断电以后会丢失信息**。非易失性存储器可以**在断电后保留信息**
- Read-only memory (ROM)：在生产的时候硬编码一些数据，一般能用20~30年（计算机启动时运行的指令集储存的地方）
- Flash memory (EEPROMs)：**允许擦除**，但约100k次后损坏
## Traditional Bus Structure Connecting CPU&Memory 
![[Pasted image 20260722115829.png|580]]
### Memory Read Transaction
执行load指令`movq A, %rax`（A是一个内存中的数据）
1. CPU将A的地址放到Memory bus上
2. Main Memory感知到这个信号后，读取该地址处8字节的内容
3. 读取的数据通过I/O bridge传送到Bus interface
4. CPU从Bus处读取数据，然后放入寄存器`%rax`

对于从CPU写入内存也是一样的，每个部分干的事是一样的
## Disk Drive
![[Pasted image 20260722153542.png|443]]
Platters由很多的Platter（盘片）组成，每个盘片涂有磁性材料（在磁性材料中编写0/1的二进制位），Arm的头浮在Platters上（薄薄一层的空气），他可以感受编码位的磁场变化。Arm由Electronics控制
### Disk Geometry
- 从z方向看
![[Pasted image 20260722162057.png|576]]
一个platter由两个面（盘面）构成——上表面和下表面，每个盘面由许多 **磁道(tracks)**（若干同心圆环）构成，每条磁道由多个 **扇区(sectors)** 组成，扇区之间由 **间隙(gaps)** 分隔（不保存数据）
- 正交于z方向看
![[Pasted image 20260722163134.png|375]]
注意这里的**Cylinder k**，多个盘面上，半径相同的所有磁道合在一起，称为一个**柱面（cylinder）**
### Disk Capacity
- Recording density：一个扇区可以存多少比特
- Track density：gaps有多小
- Areal density：上面两个的乘积
### Recording Zone
磁面上每一个磁道包含的扇区数量相等，所以越靠外的地方gaps越大，浪费的区域越多。
现代磁盘会把磁道分为若干互补重叠的子集，这些子集成为**记录区(recording zone)**
- **同一个记录区**内，每条磁道包含相同数量的扇区。这个数量由该记录区中**最内侧磁道的周长**决定
- 不同记录区中，每条磁道所包含的扇区数不同。同时我们使用**平均**的扇区数来表示扇区数

![[Pasted image 20260722173051.png|261]]
### Disk Operation
- 从z方向看
![[Pasted image 20260722173334.png|302]]
Arm会动，Platters也会动，这样Arm的头部可以读取到任意位置的信息
- 从正交于z方向看
![[Pasted image 20260722173522.png|278]]
注意，每一个surface都有一个Arm来读取
### Disk Access
![[Pasted image 20260722173739.png|170]]如果CPU需要读取这一部分的数据，Arm的头部就会在这段数据开始的地方开始读取，直到这段数据的结束，其数据会返回
![[Pasted image 20260722173938.png|166]]读取完蓝色区域以后，CPU需要读取红色区域：先要寻找红色区域的位置，然后Arm的读写头需要等待红色区域转过来读取
#### 时间组成
![[Pasted image 20260722174134.png|639]]
Seek time和Rotation latency都是实际的**机械运动**(前者约3~9ms)——这也是最大头
### Logical Disk Block
现代磁盘为复杂的扇区几何结构提供了一种更简单的抽象，所有可用扇区被建模为一个由大小为b的**逻辑块**组成的序列。这也意味着逻辑块和物理扇区有**映射关系**，其由**磁盘控制器(disk controller)** 维护——将对**逻辑块的访问转为三元组**$(\text{surface},\text{track},\text{sector})$
由于**这种抽象由磁盘控制器给操作系统**，这也就允许了磁盘控制器将一些柱面保留为**备用柱面**（他们没有被映射为逻辑块，所以操作系统正常无法访问）。如果有一个扇区坏了，磁盘控制器可以将数据复制到备用柱面，从而继续使用——这也是磁盘的**格式化容量小于实际容量**
## I/O Bus
![[Pasted image 20260722182633.png|543]]
（***注意***，这并非现代的Bus设计。上图展示的是PCI总线，因为他是**广播总线**，所以如果这根总线上的任何设备修改了某个值，该总线上的每一个设备都能看到。而现代是通过多组**点对点连接**进行连接的）
### Reading a Disk Sector
1. CPU通过编写一个**三元组**——指令、逻辑块块号、内存地址，以启动一个读取操作
![[Pasted image 20260722183115.png|371]]
2. 主存获得总线控制权，CPU此时并不知道这一情况
![[Pasted image 20260722183842.png|363]]
3. 一旦数据传入主存，磁盘控制器会使用 **中断(interrupt)** 的机制通知CPU
![[Pasted image 20260722184050.png|338]]
具体的机制是：磁盘控制器把连接到 CPU 中断输入端的一根电信号线，从低电压变成高电压（值从0到1）。这个触发器就是中断，它通知CPU这个扇区已被赋值。

这么做的原因是：在10ms内，CPU可以执行1M条指令，让CPU等磁盘传输完数据再处理会**导致极大的性能损失**。所以我们让CPU通知完硬盘后就不再管数据的传输了，再硬盘传到主存的这段时间CPU执行其他的指令，数据传输完以后再通知CPU进行相关的处理
## Solid State Disks (SSDs)
![[Pasted image 20260722200955.png]]
SSD和机械硬盘提供相同的接口，对于操作系统来说，他们是一样的
一系列的 **页(Page)** 形成 **块(Block)**（注意其和抽象里的Block是不一样的）
数据以页为单位写入，但是一个页只能在所在的整个块都被擦除后才能被写入
# Locality
**Principle of Locality：程序倾向于使用其地址接近或等于最近使用过的数据和指令的那些数据和地址**
比如一个程序访问一个数据项，那么他在不久的将来访问该数据项或附近的数据项的可能性很高
- **Temporal locality**：最近引用的储存器位置可能在不久的将来再次被引用（比如循环变量）
- **Spatial locality**：引用临近存储器位置的倾向
## Example
```C
sum = 0;
for (int i = 0; i < n; i++)
	sum += a[i];
return sum;
```
Data：对`a[i]`的引用是Spatial locality，对`sum`的引用是Temporal locality
Instruction：一系列的引用指令是Spatial locality，循环是Temporal locality
### Qualitative Estimate of  Locality
> 快速看出一个代码的局部性非常重要

```C
int sum_array_rows(int a[M][N]) {
	int i, j, sum=0;
	for (i = 0; i < M; i++) {
		for (j = 0; j < N; j++) {
			sum += a[i][j];
		}
	}
	return sum;
}
```
其具有很好的空间、时间局部性（这里主要是空间）
但是如果我们把它[[01#4|循环变量反过来]]（i在内层，j在外层），就会导致我们**循环的空间局部性很差**（一次要跳N）
# Memory Hierarchies
**硬件和软件具有一些基础且长期存在的特性：**
- 速度越快的存储技术，每字节成本越高、容量越小，并且功耗越大、发热越多。
- CPU 与主存之间的速度差距正在不断扩大。

存储器的层次结构可以利用这些特性，让专人专事
![[Pasted image 20260722214123.png]]
每一层都保存着下面一层中检索的数据
## Caches
**Cache**：其为一个更小、更快的存储设备，充当更慢的设备中的**数据暂存区**
比如可以把主存（及更高区域）认为是在磁盘上存储的数据的缓存，这样我们如果要取磁盘上的该数据，我们可以直接从主存中取出，这样会更快（上学时的背包，不必回家就可以拿书）
缓存的优势**由程序的局部性保证**——假设程序所需的数据储存在$L_{k+1}$层，如果我们将它放到$L_k$层，由于程序的局部性，在$L_k$层的数据**有更高概率会被多次访问**。**访问更高区域的次数越多，我们的性能提升就越大。**
### General Cache Concepts
假设我们原来的Cache里是`8 9 14 3`。现在CPU需要4这个数据，它会先在Cache中查找，如果查找不到，就会要求主存发送到CPU，这个时候主存就会把Cache中的一个数据覆盖为4
![[Pasted image 20260722215305.png|515]]
**Hit**：CPU需要访问的块正好位于缓存中，叫做**缓存命中**。这会非常的快
#### Type of Cache Misses
- **Cold (compulsory) miss**：Cache中没有任何数据（将数据加载到缓存里又叫做缓存暖身）
- **Conflict miss**：与缓存的实现有关：
大多数缓存（尤其是硬件缓存），由于他们必须被设计的较为简单，**限制了块可以放置的区域**（比如块号为$i$的块只能放到$(i\bmod 缓存大小)$的位置）。比如上面的例子，`0 4 8`会被放到同一个位置（被覆盖），而Cache中仍有足够的空间放置（类似于Hash冲突）
- **Capacity miss**：高速缓存有限。比如上面的例子，如果CPU需要8块数据，那么只有4块的Cache就无法满足CPU，就会导致缓存不命中
我们将一些不断被程序访问的块称为**工作集(working set)** ，一般在某个函数调用，某个循环中是不会有很大变化的。当工作集的大小超过Cache的大小，就会产生Capacity miss
### Example of Caching in the Mem. Hierarchy
![[Pasted image 20260722220823.png]]