系统如何构建程序？
# Linking
## Linking
![[Pasted image 20260825101407.png|660]]

如果在终端输入：
```bash
linux> gcc -Og -o prog main.c sum.c 
linux> ./prog
```
整个过程是：
![[Pasted image 20260825101531.png|440]]

这么做有两点好处：
- 模块化
- 效率：更改一个模块里的代码，不必要重新编译其他模块
## Linkers
链接器执行的主要任务有两个：
1. 符号解析：
   **符号定义**(`void swap();{......}`)、**符号引用**(`swap();`)——我们可以称定义的名称为**符号**
   符号定义被汇编器存储在object文件的**符号表**（一个**结构体数组**，包含这个符号的名称、大小、位置）中
   **符号解析**是指在链接过程中，链接器将每一个符号引用和**恰好一个**符号定义连接（多个模块之间的重名情况，使得Linkers需要决定到底使用哪个）
2. 重定位：
   将所有的模块**合并**为一个单一的可执行object文件
   在重定位之前，object文件里函数和数据的地址**是他们在模块中的偏移量**。Linkers需要决定每个符号最终在程序执行时位于内存的哪个位置，然后将这些**绝对内存位置绑定到符号上**。然后**更新所有的引用**，使得其指向正确的位置
### Three Kinds of Object Files (Modules)
![[Pasted image 20260825105237.png|488]]
`.o`是需要被 *链接* 以后才能执行的文件；`a.out`是可执行文件；`.so`后面讨论
### Executable and Linkable Format (ELF)
ELF是object文件的标准的二进制格式——上述的三种类型使用相同的一般格式

![[Pasted image 20260825110720.png|235]]
- ELF header：定义了如字长、字节序、文件类型(`.o`, `a.out`, `.so`)等信息
- Segment header table：只为可执行文件定义，**指示代码的不同段在内存的位置**（栈的位置，代码位置）
- .text section：代码（只读）
- .rodata section：只读数据，比如switch的跳表
- .data section：包含所有已初始化全局变量的
- .bss section：**定义**了未初始化的全局变量（不占空间）

![[Pasted image 20260825110649.png|230]]
- .symtab section：符号表
- .rel .text/.data section：当Linkers识别有对符号的引用时，他会做一个记号以告知此处需要重定位
- .debug section：储存源代码行号和机器码的行号关联的信息
- Section header table：标淡紫色的所有section的起始位置
### Linker Symbol
对于Linkers来说，有三种不同类型的符号：
- Global：没有用`static`修饰的全局变量或者函数
- External：被某个模块引用，但是在其他模块中定义的Global symbols——[[13-linking#Linking|main.c调用的sum就是在引用External symbols]]
- Local：在模块内部定义和引用的符号（注意**Linkers不知道所谓的“局部变量”**，这是由编译器在运行时栈上管理的）。这里我们指用`static`修饰的全局变量或函数

> [!NOTE] C语言的 `static`
> 1. 隐藏：
>    用`static`修饰的变量和函数会**对其他源文件隐藏**。如果不用其修饰，全局变量和函数对其他源文件可见。`static`修饰函数只有这一个作用
> 2. 静态数据：
>    被`static`修饰的变量在程序刚运行时就初始化，且只初始化一次（默认初始化为0）
#### Local Symbols
![[Pasted image 20260825153629.png|215]]
这里的每一个静态变量`x`都只能在其定义的函数内作用。此外，`x`被`static`修饰以后，不会再被储存在栈中——而是.data（编译器会**在.data里声明空间**，并且给同名的变量取不一样的名字）
#### Example
![[Pasted image 20260825153427.png|625]]
#### Duplicate Symbol Definitions
如何解决符号重名的问题？我们先把符号分为两种：
- 强符号：函数名或已初始化的全局变量
- 弱符号：未初始化的全局变量
![[Pasted image 20260825155234.png|605]]
#### Linker’s Symbol Rules
![[Pasted image 20260825155409.png|593]]
**规则3可能引发问题**，`-fno-common`会使得出现多个弱符号时Linkers报错
### Linker Puzzles
![[Pasted image 20260825160548.png]]注意这里的**overwrite**——如果程序一次在写入的时候选择了那个`double x`，由于`double x`是8字节，在写入(`x=1`)的时候有可能写到