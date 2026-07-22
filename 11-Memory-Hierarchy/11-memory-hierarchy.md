# Storage technologies and trends
系统提供了一个抽象，把内存结构抽象为一个大的线性数组
## Random-Access Memory (RAM)
- RAM被包装为芯片，多个RAM芯片组成**主存**
- RAM的基本存储单元称为**单元**(cell)，大小为1bit
- RAM分为两种类型：**S**RAM(Static)、**D**RAM(Dynamic)
![[Pasted image 20260722113843.png|577]]
refresh指以一定电压冲入电荷，如果DRAM没有refresh就会丢失电荷，导致丢失信息；EDC指Error Detaction&Correction，可以看出SRAM会比DRAM更可靠
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
具体的机制是：磁盘控制器把连接到 CPU 中断输入端的一根电信号线，从低电压变成高电压