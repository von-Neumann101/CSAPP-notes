`%rsp`指向一个地址
# sscanf
这里传给 `sscanf` 的**输入一定是字符串**，解析结果存入 6 个 `int` 变量中。
sscanf的原型是：
```C
int sscanf(const char *str, const char *format, ...);
```
注意这也是一个函数，所以也要注意传入参数存入的寄存器
![[Pasted image 20260529165408.png|278]]

| 参数        | 寄存器/栈  | 含义             |
| --------- | ------ | -------------- |
| 第 1 个参数   | `%rdi` | 输入字符串 `str`    |
| 第 2 个参数   | `%rsi` | 格式字符串 `format` |
| 第 3 个参数   | `%rdx` | 第 1 个整数的地址     |
| 第 4 个参数   | `%rcx` | 第 2 个整数的地址     |
| 第 5 个参数   | `%r8`  | 第 3 个整数的地址     |
| 第 6 个参数   | `%r9`  | 第 4 个整数的地址     |
| 第 7、8 个参数 | 栈上     | 第 5、6 个整数的地址   |
对应的汇编为
```asm
sub    $0x18,%rsp                       
mov    %rsi,%rdx                        # rdx = numbers，即 &numbers[0]
lea    0x4(%rsi),%rcx                   # rcx = &numbers[1]
lea    0x14(%rsi),%rax
mov    %rax,0x8(%rsp)                   # 栈上 = &numbers[5]
lea    0x10(%rsi),%rax                  
mov    %rax,(%rsp)                      # 栈上 = &numbers[4]
lea    0xc(%rsi),%r9                    # r9 = &numbers[3]
lea    0x8(%rsi),%r8                    # r8 = &numbers[2]
mov    $0x4025c3,%esi                   # rsi = 格式字符串地址
mov    $0x0,%eax
call   0x400bf0 <__isoc99_sscanf@plt>
cmp    $0x5,%eax
jg     0x401499 <read_six_numbers+61>
call   0x40143a <explode_bomb>
add    $0x18,%rsp
ret
```
在调用读数函数之前`%rsp`已经-`0x50`了，这是初始化，我们可以不管——我们只考虑函数调用里的栈变化：
```
%rdx       ···0       原%rsp: 输入整数数组的首地址
%rcx       ···1       原%rsp + 0x4
%r8        ···2       原%rsp + 0x8
%r9        ···3       原%rsp + 0xc
---        ···stack↓
(%rsp)     ···4       原%rsp + 0x10 ->存在原%rsp - 0x20
0x8(%rsp)  ···5       原%rsp + 0x14
```
一定要区分——`%rsp`和`(%rsp)`
**当前`%rsp`指向的内存里，存放着 `原%rsp+0x10` 这个地址值**
```
地址            内容
S - 0x20        S + 0x10      ← 当前 %rsp 指向这里；里面放的是 &numbers[4]
S - 0x18        S + 0x14      ← 里面放的是 &numbers[5]

S + 0x00        numbers[0]
S + 0x04        numbers[1]
S + 0x08        numbers[2]
S + 0x0c        numbers[3]
S + 0x10        numbers[4]
S + 0x14        numbers[5]
```

