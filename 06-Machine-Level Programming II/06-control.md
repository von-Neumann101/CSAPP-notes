# Control: Condition codes
## Processor State
寄存器`%rsp`是栈指针（后面会讲）
寄存器`%rip`指向**下一条即将被执行的指令的地址**
**Status of recents tests(Condition codes)**: CF, ZF, SF, OF......，他们是由**其他指令运行结果**设置的1位flag
## Condition Codes
- CF(Carry Flag) 进位（无符号）
- ZF(Zero Flag) 运算结果为0，这个位会被置1
- SF(Sign Flag) 负数——置1
- OF 有符号数溢出

**显式设置Condition Codes**
- `cmpq b, a`，计算a-b但是只设置Condition Codes，不改变其他结果
![[Pasted image 20260512191625.png|572]]
- `addq a, b`，计算t=a+b
![[Pasted image 20260512191648.png|495]]
- `testq b, a`，计算a&b
![[Pasted image 20260512191910.png|213]]
如果满足则置1，否则置0

## Read Condition Codes
读取、使用
### SetX指令
将单个寄存器的单个字节设置为1或0，判断设置0还是1基于条件码的值（在他之前最近的指令做了什么），Set指令有多种，其置1的条件不同
![[Pasted image 20260512192514.png|500]]
所有寄存器的最低位字节可以设置为0或1，而且不影响这个寄存器内其他7个字节。
## Example
![[Pasted image 20260512192926.png]]![[Pasted image 20260512192935.png|255]]
机器代码要做什么呢？——当x>y返回1，也就是把%rax设置为1，也就是把第一个字节置为1，其他字节置为0

![[Pasted image 20260512200647.png|515]]
这里的`%al` 是 `%rax` 的最低 8 位（最低字节），`%eax`则是低32位
`movzbl`是把 **1 字节**的数据搬到 **4 字节寄存器**里，并且高位补 0
**x86-64的任何计算结果是32位，然后把其余32位设置为0**。这也是为什么movzbl只需要把低32位后面变成0（高32位已经为全0了）
# Condition Branch
## Jumping
一行一行地执行语句，jump可以随意地跳转到任何地方
![[Pasted image 20260512202054.png|506]]

![[Pasted image 20260512202221.png|224]]![[Pasted image 20260512202234.png|343]]
汇编语言为：
![[Pasted image 20260512202251.png|297]]
jmp有直接跳转和间接跳转：
- 直接跳转：直接通过标签跳转
- 间接跳转：使用`*`将操作数中的值作为跳转目标
### Goto Code
使用了goto的C语言代码：
![[Pasted image 20260512202753.png|266]]
## Conditional Moves
把if-else两个结果都计算一遍，然后决定使用哪个结果（实际上效率很高，以后会说。由于流水线技术，还有分支预测技术，这是对于计算机的好的结果）
> [!NOTE] 补充
> 处理器使用**流水线**技术提高效率。在流水线中，一条指令的处理要经过一系列阶段，每个阶段执行所需操作的一小部分（执行算术运算，内存读数据......）。其通过**重叠连续指令的步骤**获得高性能，例如，取一条指令的时候，执行其前面一条指令的算术运算——所以要要求事先确定指令序列，**以保持流水线满功率运作**。
> 遇到条件分支，就出现了一个问题——一定要把条件算出来才能确定序列，为了解决这个问题，处理器使用分支预测技术来猜测会去哪里。显然，错误的猜测会导致程序性能严重下降。

![[Pasted image 20260512204016.png|228]]
### 缺点
![[Pasted image 20260512204242.png|477]]
- 分支的计算量都很大
- 用于防止危险操作的分支，如果两个分支都计算则失去效果
- 会对后续程序运行产生影响的（副作用）的
GCC只有在两个分支都是简单直接计算的时候才会使用conditional move优化
# Loops
## Do-While Loop
![[Pasted image 20260512204536.png|335]]
很好理解
## While Loop翻译方式
- jump to middle
```C
	goto test;
loop:
	body_statement
test:
	t = test_expr
	if (t)
		goto loop;
```
- guarded-do
```C
t = test_expr
if (!t)
	goto done;
do
	body_statement
	while (test_expr);
done:
```
# Switch Statement
![[Pasted image 20260512210725.png|312]]
机器码并不是像C语言中if-else叠加实现switch的
## Jump Table Structure
![[Pasted image 20260512211017.png|595]]
Jump Table是一个数组，表项i是一个**代码段的地址**，这个代码段实现当 *开关索引* 值为i时程序应该采取的动作。
相比于叠加if-else相比，Jump Table跳转所需的时间**和开关数量无关**（这是自然的，因为一个需要判断每一个`test_expr`，而Jump Table只需要像访问数组一样）
## Example
![[Pasted image 20260512211401.png|237]]
示例代码的汇编：
![[Pasted image 20260512211420.png|405]]
之所以比较6，是因为case中最大的为6
`ja`很聪明——因为这是无符号比较，所以负数（大于所有正数）和大于6的数都会到default里
注意`jmp`处的地址表达式，就是对表进行索引，并提取一个地址并跳转(Core)

**jump table会建立从最大到最小每一个的盒子，没有的case使用default来填充**
![[Pasted image 20260512212950.png]]
如果值范围很大且稀疏，则会使用if-else树（二分）
同时通过偏置，可以把最小值过大或过小纠正
[[06-control#Switch|补充（来源于书）]]
## Handling Fall-Through
编译器会把没有break(Fall through)的代码块融合在一起
## 优点
在switch语句中获取所需位置的时间复杂度是O(1)
![[Pasted image 20260512213443.png]]
注意这里是直接算地址然后跳转过去，而不是一个个遍历然后查找！
# 补充
## `rep`和`repz`
和`ret`组合(`repz retq`)用于防止跳转到`ret`，用于使得代码在AMD上运行速度更快，此外**不会改变任何代码行为**
## 机器代码
```object
0: 48 89 f8        mov   %rdi,%rax
3: eb 03           jmp   8 <loop+0x8>
5: 48 d1 f8        sar   %rax
8: 48 85 c0        test  %rax, %rax
b: 7f f8           jg    5 <loop+0x5>
d: f3 c3           repz retq
```
注意这里jump指令后面的数字（这里数字都是补码表示）
```
对于第二个指令：0xf8(目标编码第二个字节) + 0xd(下一条指令的地址) = 0x5(跳转到的地址)
第一个指令：0x03 + 0x5 = 0x8
```
跳转到的地址（相对）为**跳转指令目标编码的第二个字节+下一条指令的地址**
当然，链接过后的地址变为绝对地址，则是：**跳转目的地的绝对地址=跳转指令目标编码的第二个字节+下一条指令的绝对地址**
## C语言指针（单纯因为我有点忘了）
```C
int x = 0;
int* p = &x;
```
从Java中变量的申明来看，我们申明的不是`int`类型的`*p`而是`int*`类型的`p`，完全可以把指针当做一个存储地址的整形(64 bits)，而`*`是一个运算符，用于在这个地址出取元素
在上面的代码中，我们申明`p`储存`x`的地址，自然地，`*p`就的值`x`
## Switch
![[Pasted image 20260530155911.png|640]]
`&&`运算符创建一个**指向代码位置**的指针
注意看b)的`jt[7]`处，`&&loc_def`出现了两次——这是因为100到102中的101掉了，**要把这个case归为default**，如果从100直接到103，该处就会有两个`&&loc_def`
这里的一个小巧思是：index被定义为unsigned long，所以当n-100小于0的时候会溢出，导致index>6
汇编版本：
![[Pasted image 20260530161827.png|471]]![[Pasted image 20260530162055.png|426]]
`.L4` 通常位于只读数据区，这里就是用于Jump Table
注意这里使用的是跳转是`ja`，这是无符号比较
C语言的`break`这里等价为`jmp`