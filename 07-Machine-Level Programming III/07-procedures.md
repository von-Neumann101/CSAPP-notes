*ABI——应用程序二进制接口（课程使用的约定）*
**过程**：提供了封装代码的方式，用一组指定的参数和一个可选的返回值实现了某种功能，使得程序可以在不同地方调用他（过程的形式包含但不限于：函数、方法、子例程、处理函数）

![[Pasted image 20260519082517.png|484]]
# Stack Structure
如何将控制传递给函数，栈的结构适合处理**调用**
## x86-64 Stack
`%rsp`寄存器永远指向栈顶（内存小端），故其值为当前栈顶的地址
通过递减该指针，我们可以填入数据（扩大栈）
![[Pasted image 20260519083251.png|250]]
## push & pop
`pushq Src`通过递减栈顶指针，然后在新的位置（存在`%rsp`中）写入对应数据
`popq Dst`递增栈顶指针，然后把原来的栈顶存在Dst（由于内存不能直接到内存，所以Dst只能是寄存器）
# Calling Convention
## Passing Control
![[Pasted image 20260519084644.png|572]]
非常巧妙的一点是——一个过程如果执行完了，他的栈帧就为空了。在栈里面就非常适合在P调用完过程Q以后，栈指针处于P的帧处。
![[Pasted image 20260602190157.png|320]]
### Procedure Control Flow
- Procedure call: `call label`把返回地址存入栈，然后jump到label处
- Procedure return: `ret`弹出栈的地址，然后jump到该地址
#### Example
![[Pasted image 20260519085604.png|472]]

调用`callq`以后：
![[Pasted image 20260519085710.png]]
把`callq`这条指令执行完以后下一条应该执行的命令的地址push入栈
同时，把call的目标的地址存入`%rip`（指向**下一条即将被执行的指令的地址**）寄存器（这也是必须的）

调用`retq`以后：
![[Pasted image 20260519090812.png]]
和`call`完全相反
## Passing data
### Procedure Data Flow
ABI规定：函数传入的前六个参数需放在如下的六个寄存器中，如果超过六个则压入栈中。返回的值放入`%rax`中
![[Pasted image 20260519091156.png|514]]
## Managing local data
### Stack-Based Languages
把栈上用于特定call的每个内存块称为**栈帧**
![[Pasted image 20260519092936.png|504]]
### Stack Frame
内容
- Return information(`call`, `ret`)
- Local storage(局部变量)
- Temporary space
栈帧的管理就是由上述的`call`和`ret`完成
一般情况，机器知道要分配多少的空间，进而也知道要清除多少的空间（当不在需要这些空间内的元素时）。但有的时候机器并不知道他需要分配多少的空间，这个时候会分配一个可变大小的数组或内存缓冲区（此时`%rbp`是必须的）
#### Example
![[stack_frames_from_pdf_hd.webp|522]]
所以无限的递归导致栈错误
#### x86-64 Stack Frame
![[Pasted image 20260519203008.png|239]]
这里Saved Registers指的是**当前函数把某些寄存器的旧值保存到自己的栈帧里，以便之后恢复**。这是由于*之前罗列的传入参数必须放入的寄存器*内部原来就有值，这些值需要在函数调用结束以后恢复，而这里就是保存原始值的地方
#### Example
![[Pasted image 20260520084622.png|265]]![[pages_34_38_hd.webp|572]]
1. 先扩大栈（`%rsp`减16），然后在栈底加入数据
2. 传入参数
3. 调用函数
4. 将结果存在`%rax`并清空栈
5. `ret`
编译器会计算出需要的空间，栈并不是用了某种奇妙的规则，而是在完全保证规则的情况下硬编码运行
### Register Saving Convention
在调用`call`以后，原先保存在某些寄存器中的值可能会被覆盖
约定：
- caller在调用前保存的都是**临时**的值
- callee把临时值保存在自己的帧里，但在return的时候恢复他们
### x86-64 Linux Register Usage
#### 1
![[Pasted image 20260520092040.png]]
Caller-saved的意思是——**Caller**在调用函数以后，还需要某个寄存器的旧值，Caller在调用之前就需要**把它们保存起来**
#### 2
![[Pasted image 20260520092400.png]]
can mix & match的意思是`%rbp`有多重用法——Frame Pointer或者是普通的Register
### Callee-Saved Example
![[Pasted image 20260520093507.png|344]]
因为接下来要把`%rdi`的值传入`%rbx`
![[Pasted image 20260520094823.png|120]]![[Pasted image 20260520094842.png|96]]
# Recursion
```C
long pcount_r(unsigned long x) { 
    if (x == 0) return 0; 
    else return (x & 1) + pcount_r(x >> 1); 
}
```
![[Pasted image 20260520095303.png|267]]
由于[[Lecture 6 Recursive Function|栈的特性]]，无需特殊考虑递归函数的情况
# 补充
## 练习题
![[Pasted image 20260603192939.png]]
由`sizeof(a) + sizeof(b) = 6`得到`a`和`b`中有一个int，有一个short
- 为何能从第二句判断出`%edi`存入的是int类型？
`movslq`是做符号拓展的`mov`指令，注意其Src是低32位，而short类型是16位。对于负short类型而言，如果`%edi`的高16位**没有正确地拓展符号位**，会得到**错误结果**（没有填0xF...F而是0x0...0）。虽然，理论上**我们自己写汇编**完全可以这么写（这在语法上没有错误），但这里是编译器输出的结果，这是没理由这么写的，所以完全可以确定`a`为int类型，`b`为short类型
- 为何能从第四句判断出`%rcx`指向的是char类型数据？
切记除了像`movsql`这种指令，其他的指令都要求Src和Dst的长度一致，所以如果用了内存寻址语句，就**隐藏了该内存位置数据的类型**，比如这里`addb %sil, (%rcx)`。之所以能判断，是因为`addb`要求Src和Dst的位长都是b
