编译器在对有些代码编译的时候不会选择优化，本节主要阐述如何编写“编译器友好”代码
# 常用优化
## 代码移动
以下是用于替换二维数组整行的函数
![[Pasted image 20260613153820.png]]
可以只计算1次`n*i`，而不是n次（如果gcc优化等级为1或更高，编译器会自动优化）
## 减少强度
- 使用更有效率的操作（使用移位和加减而非乘除）
加i次n比`i*n`快：
![[Pasted image 20260613154447.png|571]]

- 通过复用表达式减少低效率运算：
![[Pasted image 20260613154705.png]]
> [!NOTE] 空间&时间trade-off
> 对于现在的计算机来说，时间的重要性远远高于空间

# Optimization Blocker
有些代码无论是什么编译器都无法编译的
## Procedure Calls
```C
void lower(char *s) { 
	size_t i; 
	for (i = 0; i < strlen(s); i++) 
		if (s[i] >= 'A' && s[i] <= 'Z') 
			s[i] -= ('A' - 'a'); 
}
```
![[Pasted image 20260613184433.png]]
换为goto的形式：
```C
void lower(char *s) { 
	size_t i = 0; 
	if (i >= strlen(s)) 
		goto done; 
 loop: 
	if (s[i] >= 'A' && s[i] <= 'Z') 
		s[i] -= ('A' - 'a'); 
	i++; 
	if (i < strlen(s)) //this!
		goto loop; 
 done: 
} 
```
主要的原因是：`strlen(s)`的时间复杂度是$O(n)$，然后for循环$n$次，然后每次循环都会调用这个复杂度为$O(n)$的函数，所以总的就是$O(n^2)$平方时间

所以我们提前`int n=strlen(s)`就可以了
![[Pasted image 20260613185455.png]]

有几个原因使得编译器无法优化：
1. 字符串在循环中被修改了，编译器没理由觉得strlen不会变
2. 由于可能有多个版本的strlen，编译器不知道其是否有副作用（不能假设没有副作用）
## Memory Matters
```C
/* Sum rows is of n X n matrix a and store in vector b */ 
void sum_rows1(double *a, double *b, long n) { 
	long i, j; 
	for (i = 0; i < n; i++) { 
		b[i] = 0; 
		for (j = 0; j < n; j++) 
			b[i] += a[i*n + j]; 
	} 
}
```
对应的汇编代码：
![[Pasted image 20260613190540.png|547]]
在循环中，每一个元素都需要——“从内存中取b的元素，放`%xmm0`中，从内存中取a的元素，加到`%xmm0`中，再把`%xmm0`存回b的内存中“，为何？
**Memory Aliasing**：
如果b为a中的某一行，比如：
```C
double A[9] = 
  { 0, 1, 2,
    4, 8, 6, 
    2, 4, 8}; 
double* B = A+3;
```
这种现象称为**别名(Aliasing)**——即**程序中的不同部分指向内存中的相同区域**。
编译器需要大量工作检查是否有内存别名引用，不过编译器**一般假设存在内存别名引用**（也就是假设二者有内存重叠），所以编译器会把值写到内存中然后一遍一遍地读
**Solution**：
![[Pasted image 20260613202736.png|523]]

## 利用并行性
简单的程序变换可以带来显著的性能提升，编译器通常无法自动完成这些变换，原因之一是：浮点数运算中不满足严格的结合律和分配律。
### CPE
这里我们使用**CPU的时钟周期**来度规时间，Cycles Per Element(CPE)
![[Pasted image 20260613194040.png|659]]
Overhead就是固定开销，比如size=0的情况；Slope就是CPE，表示处理一个元素平均需要多少个 CPU 时钟周期
### Modern CPU Design
![[Pasted image 20260613200016.png|615]]
CPU尽可能多地读取指令序列，然后CPU把读入的指令拆开，发现有的指令不是相互依赖的，CPU就可以直接执行后面的语句（由于指令的独立性）——**指令级并行性**
总结：把程序的操作拆分然后重组，使得基本单元都在工作
#### Supersclar Processor
**超标量指令处理器**：在一个时钟周期，可以同时进行多项操作的CPU
#### Pipelined Functional Units
现代CPU采用**乱序执行**，其中功能单元使用了**流水线技术**——将计算分解为一系列不同的阶段
```C
long mult_eg(long a, long b, long c) { 
	long p1 = a*b; 
	long p2 = a*c; 
	long p3 = p1 * p2; 
	return p3; 
}
```
这张表就是流水线技术的体现（表示了一个乘法功能单元内部被分成 3 个 stage），表往下是计算的阶段（流水线拆的——计算乘法要三个步骤），往右是计算的时间（或者说步骤）
![[Pasted image 20260613201340.png|531]]
```
第 1 个 cycle：
a*b 进入乘法器的 Stage 1

第 2 个 cycle：
a*b 被推进到 Stage 2
a*c 同时进入 Stage 1

第 3 个 cycle：
a*b 被推进到 Stage 3
a*c 被推进到 Stage 2

第 4 个 cycle：
a*b 完成
a*c 被推进到 Stage 3

第 5 个 cycle：
a*c 完成
p1*p2 才能进入 Stage 1 (p3依赖于p1,p2，必须等他们做完了才能继续算)
```
#### Haswell CPU
![[Pasted image 20260613203100.png]]
每个指令有两个参数：
- Latency：是一个指令从头到尾执行的时间
- Cycles/Issue：同一个功能单元多久可以接收一条新的同类指令，即**连续两条指令进入流水线的最小时间间隔**
可以看到，除法几乎是没有流水线技术优化的，这是非常昂贵的操作。
### Example
```C
/* data structure for vectors */ 
typedef struct{ 
	size_t len; 
	data_t *data; 
} vec;
```
`data_t`由宏定义
```C
/* retrieve vector element and store at val */ 
int get_vec_element(*vec v, size_t idx, data_t *val) { 
	if (idx >= v->len) //越界检查
		return 0; 
	*val = v->data[idx]; 
	return 1; 
}
```
#### Navie
![[Pasted image 20260613194027.png|540]]
![[Pasted image 20260613194854.png|532]]
- 对同一个数组多次检查是否越界(`get_vec_element`)
- 直接在循环进行[[10-optimization#Memory Matters|数组操作]]
- 每次循环都调用一遍`get_vec_element`
#### Basic Optimization
![[Pasted image 20260613195744.png|448]]
`combine4`内部循环对应的汇编
![[Pasted image 20260613203657.png|536]]
（回顾[[10-optimization#Haswell CPU|Latency]]）
![[Pasted image 20260613203709.png|481]]
由于是累乘，我只有知道上一次乘法的结果才能继续算下去，恰好对应Latency Bound
#### Reassociation
即使我有[[10-optimization#Pipelined Functional Units|流水线技术]]，由于我的程序，乘法运算还是只能逐项进行
![[Pasted image 20260613204018.png|581]]
要是能提前知道全部`d0,d1,...,d7`就好了，这样乘法就是独立的，可以使用流水线技术
##### Loop Unrolling
