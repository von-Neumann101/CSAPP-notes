# Arrays
机器代码里没有数组的抽象
## Array Allocation
![[Pasted image 20260524111703.png|673]]
决定大小的有——存储数据的类型和个数
## Access
![[Pasted image 20260524112006.png|615]]
指针就是存储地址一段内存，指针和数组是**等价**的（但是需要注意的一点是——数组名这个指针**始终指向数组的第一个元素，无法更改**）
所以对于不同数据类型的数组，其指针自增所增加的量**不同**（和数据类型有关）
比如`char *pc`和`int *pi`
```
pc++ <=> pc指向的地址加1
pi++ <=> pi指向的地址加4
```
同时，C语言数组的语法糖
```
val[x] <=> *(x + val) <=> &val + x * sizeof(val[0])处的元素
```
这也表面**允许负数索引**，但和Python中的循环不同，C语言是**直接访问数组以外的区域**
## Example：数组遍历
![[Pasted image 20260525203407.png]]
## 指针 & 数组
以下是上课展示但课件没有的部分：
![[Pasted image 20260525204627.png]]![[Pasted image 20260525204633.png]]
![[Pasted image 20260525204639.png]]![[Pasted image 20260525204645.png]]
当我们看到一个指针，我们想知道：
1. 这合法吗？（能否编译）
2. 这可能给一个空指针引用吗？
3. sizeof的结果是？
这体现了数组和指针的区别：
创建数组的时候，我们既**分配空间**也创建一个**用于指针运算的数组名**
```
int (*A3)[3] -> int[3] (*A3) -> A3是一个指针，指向一个长度为3的int类型数组
```
## 高维数组
以二维数组举例
```C
int A[R][C];
```
![[Pasted image 20260526092001.png]]
**依次连续展开**
虽说是二维数组，实际上，`A[R][C]`是一个R元素数组，其中的元素是C元素数组。所以，`A[i]`代表的就是数组，起始地址为`A + i * (C * sizeof(A[0][0]))`
对于其中的int元素，其地址为`A + (i * C + j) * sizeof(A[0][0])`
### Example
这是一个很好的例子
![[Pasted image 20260526093149.png]]
![[Pasted image 20260526093202.png]]
## 变长数组
允许数组的维度是表达式，**在数组被分配的时候才计算出来**
我们经常写的
```C
int n;
scanf("%d", &n);
int A[n];
```
或者
```C
int func(int n, int A[n][n], int i, int j) {
    return A[i][j]; //编译器知道用i * n + j来索引 
}
```
![[Pasted image 20260610152106.png|565]]
注意到这里使用乘法而非移位+offset，这会导致性能降低
## 访问元素
![[Pasted image 20260526100845.png]]
注意，由于这里的**内存不是连续的**，虽然C代码一样，但是汇编代码不同
```asm
Mem[pgh+20*index+4*digit] 
Mem[Mem[univ+8*index]+4*digit]
```
# Structures
## 简介
对C语言结构体很熟的可以跳过，这里用作复习
C语言的结构体和Java的类比较相似，比如下面的对于矩形的定义：
```C
struct rect {
    long x;
    long y;
    unsigned long w;
    unsigned long h;
    unsigned color;
};
```
我们便可以申明一个`struct rect`类型的变量
```C
struct rect r;
r.x = r.y = 0;
r.w = 10;
r.h = 20;
r.color = 0xFF00FF;
```
或者也可以
```C
struct rect r = {0, 0, 10, 20, 0xFF00FF}; //保证一一对应
```
通常（几乎）我们都使用指针来传递结构体（这么做是由于C语言是[[L3 Reference Recursion intList#^342a3d|值传递]]的）
```C
long area(struct rect *rp) {
    return (*rp).w * (*rp).h;
}
```
由于过于常用，C有一个简化写法`rp->w`
## Overflow
![[Pasted image 20260526104256.png]]
由于没有边界检查，这种连续内存访问会导致[[09-advanced#Buffer Overflow|严重错误]]
## 链表
![[Pasted image 20260526104645.png|276]]
其汇编代码：
![[Pasted image 20260526105455.png]]
> [!NOTE] `->`
> 先把结构体指针 `r` 解引用，得到它指向的那个 `struct rec` 结构体对象，然后访问其中的成员 `i`
## Structures & Alignment
***注意：对齐的永远是地址***
![[Pasted image 20260526105940.png|539]]
编译器在实际分配空间的时候会在数据中插入一些空白数据，以使得其对齐。
注意这里K是**某个数据对象的对齐要求**（具体看后面）
### Alignment Principles
大多数机器一次取**64字节**。一般来说，如果没有一个**对齐的地址**，一个特定的数据跨过两个块时，会让硬件或者操作系统来采取额外步骤来处理（对于x86来说会导致效率变低，而对于某些机器甚至会导致错误）

### Specific Cases of Alignment(x86-64)
如果一个地址是$2^k$的倍数，他的低k位一定全为0
![[Pasted image 20260526110600.png|476]]
以short(2 Byte)举例
```
地址
0x1000 可以放   -> 0x0 = 0000
0x1001 不可以放 -> 0x1 = 0001
0x1002 可以放   -> 0x2 = 0010
0x1003 不可以放 -> 0x3 = 0011
```
很好理解，实际上就是要求k字节的数据的起始地址要为k的倍数，而k(二的幂)的二进制表示的低log k位为0。

结构体的**起始地址必须是K的倍数**（**在结构体中**，K代表其中所有成员的最大对齐要求），同时，结构体大小**必须是K的倍数**：
![[Pasted image 20260527081925.png|630]]
## 节省空间
![[Pasted image 20260527082336.png]]最大的数据放开头——注意，编译器并不会自动设置
# Floating Point
使用`%xmm`寄存器`%xmm0,xmm1,......`，返回值放在`%xmm0`中
