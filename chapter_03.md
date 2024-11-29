
# 第三章  程序的机器级表示

## 3.2 程序编码

假设有一个c程序，有两个文件p1.c和p2.c。可以使用一下命令编译：

```sh
gcc -Og -o p p1.c p2.c
```

> - -Og:使用生成符合原始c代码整体结构的机器代码优化等级。

### 3.2.1 机器级代码

对于机器级编程，两种抽象非常重要：

- `指令集体结构或者指令集架构（Instruction Set Architecture ,ISA）`:定义机器级程序的格式和行为。它定义了处理器状态，指令的格式，以及每条指令对状态的影响。
- **机器级程序使用的内存地址是虚拟地址。提供的内存模型看上去是一个非常大的字节数组。**

编译器编译的过程是将C语言提供的相对抽象的执行模型表示的程序转化为处理器执行的非常基本的指令。

汇编代码表示非常接近机器代码，与机器代码的二进制格式相比，汇编代码的特点是`它用可读性更好的文本格式表示。`

机器代码用到的寄存器有：

- **程序计数器PC（`%rip`表示）**：给出将要执行的下一条指令在内存中的地址。
- **整数寄存器**：包含16个命明的位置，分别存储64位的值。
- **条件码寄存器**：保存最近执行的算数或逻辑指令的状态。比如用来实现if'和while语句。
- **一组向量寄存器**：存放一个或多个整数和浮点数。


### 3.2.2 生成机器代码示例

假设有一个c代码文件mstore.c。生成汇编代码的方式有两种：

- 方式1：命令行使用`-S`选项可以生成汇编代码：

```
gcc -Og -S mstore.c
```

- 方式2：使用`-c`选项，GCC编译并汇编该代码，并生成.c文件对应名称的.o目标文件。

```
gcc -Og -c mstore.c
```

> -c对应的汇编过程将编译后的汇编代码转换成机器码。如，上述指令会生成一个mstore.o的目标文件，他是二进制格式无法查看。
>
> 对于二进制文件我们可以使用OBJDUMP反汇编器（disassembler）来进行反汇编。、
>
> ```
> objdump -d mstore.o
> ```

以下是对同一个c文件使用两种方式得到的汇编代码的示例：

- 方式1：使用gcc直接编译得到汇编代码

```assembly
        .file   "exchange.c"
        .text
        .globl  exchange
        .type   exchange, @function
exchange:
.LFB0:
        .cfi_startproc
        endbr64
        movq    (%rdi), %rax
        movq    %rsi, (%rdi)
        ret
        .cfi_endproc
.LFE0:
        .size   exchange, .-exchange
        .ident  "GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0"
        .section        .note.GNU-stack,"",@progbits
        .section        .note.gnu.property,"a"
        .align 8
        .long   1f - 0f
        .long   4f - 1f
        .long   5
0:
        .string "GNU"
1:
        .align 8
        .long   0xc0000002
        .long   3f - 2f
2:
        .long   0x3
3:
        .align 8
4:

```

- 使用objdump反汇编二进制机器程序得到的汇编代码：

```assembly

exchange.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <exchange>:
   0:   f3 0f 1e fa             endbr64
   4:   48 8b 07                mov    (%rdi),%rax
   7:   48 89 37                mov    %rsi,(%rdi)
   a:   c3                      ret

```

这里有几点需要注意：

- **·**  开头的行都是指导汇编器和链接器工作的伪指令。我们可以忽略这些行。
- x86-64的指令长度从1-15个字节不等，常用的指令占较少字节，不常用的占较大字节。
- 设计指令格式的方式是，从某个给定位置开始，可以将字节唯一的解码成机器指令。例如：pushq %rbx是以字节值53开头。mov    (%rdi),%rax 是以 48开头
- 反汇编器使用的指令命名规则与GCC生成的汇编代码使用的有些些细微的差别。例如：会省略某些指令的结尾，比如上述示例的 ‘q’。

## 3.3 数据格式

有是从16位体系结构扩展成32位的 ，intel用属于 `字（word）`表示16位数据类型。32位为双字，64位为四字。

C语言数据类型在x86-64中的大小：

![3.1](chapter03.assets/image-20241118093723320.png)


## 3.4 访问信息

一个x86-64的中央处理单元（cpu）包含一组16个存储64位值的`通用目的寄存器`。这些寄存器用来存储**整数数据和指针**。

![3-2](chapter03.assets/image-20241115091259052.png)

当这些寄存器作为目标时，对于生成小于8字节结果的指令，寄存器中剩下的字节操作有两条规则：

- 生成1和2字节数的指令会保留剩下字节不变。

- 生成4字节的指令会把高位4个字节置为0.

  > 该规则作为从IA32到x86-64的扩展的一部分而采用的。

上述寄存器中最特殊的是栈指针`%rsp`，用于指明运动时栈的结束位置。



### 3.4.1 操作数指示符

指令一般有一个或多个操作数，用于指定指令执行操作需要使用到的源数据以及放置结果的目的位置。

操作数有三种类型：

1. **立即数（immediate）** 用来表示常数

   格式：$后面跟一个C表示法表示的整数

   ```
   $-577
   $0x172f
   ```

2. **寄存器（register）**表示某个寄存器的内容

   通常用r~a~来表示任意寄存器a，用 **R[r~a~]** 来表示它的内容。

3. **内存引用** ：会更具计算出来的地址（通常称为有效地址）访问某个内存内容。

   用符号 **M~b~[Addr]** 表示对存储在内存中地址Addr开始的b个字节值的引用。

有多种不同的寻址模式，下表列出了所有的操作数格式：

![3-3](chapter03.assets/image-20241115090945700.png)

> 注：比例因子s必须是1、2、4或者8

### 3.4.2 数据传送指令

数据传送指令的作用是将数据从一个位置复制到另一个位置，使用最频繁。

最简单形式的传送指令-MOV类。由四条指令组成：movb、movw、movl和movq。

他们执行的操作相同，主要区别是操作数的大小分别对应： 1,2,4和8字节

语法格式：

```
MOV  S,D       //将S传送到D。
			 // S-源操作数，D目的操作数。
```

注意：

- 源操作数指定的值是一个立即数。存在寄存器或者内存中。
- 目的操作数指定一个位置（寄存器或者内存）
- X86-64限制：传送的两个操作数不能都指向内存位置。原因：这种内存内复制的操作需要两条指令实现：①加载到寄存器；②将寄存器的值复制到内存指定位置。

- 除了movl，其他指令只更新操作数指定的寄存器字节或内存位置。movl指令以寄存器为目的时，会将寄存器的高位4字节设置为0.（类似于自动实现0扩展）

MOV指令源和目的类型的五种可能组合：

| 指令                    | 说明          |
| ----------------------- | ------------- |
| `movl $0x4255 ,%eax`    | 立即数—寄存器 |
| `movw $-17,(%rsp)`      | 立即数—内存   |
| `movw %bp,%sp`          | 寄存器—寄存器 |
| `movq %rax,-12(%rbp)`   | 寄存器—内存   |
| `movb (%rdi,%rcx)，%al` | 内存—寄存器   |

movq指令只能以表示32位补码数字的立即数作为源操作数。而后把这个值符号扩展到64位的值，放到目标位置。想要实现以任意64位立即数作为源操作数，需要使用movabsq（传送绝对四字指令）。

注意：**movabsq** 只能以寄存器作为目的。

![3-4](chapter03.assets/image-20241114195803901.png)

> #### 理解数据传送如何改变目的寄存器
>
> 关于数据传送指令是否以及如何修改目的寄存器的高位字节有两种不同方式，如下图所示：
>
> ![p124](chapter03.assets/image-20241114200057103.png)

MOVZ类中的指令目的操作数寄存器中剩余字节填充0，而MOVS类中的指令通过符号扩展来填充。

![3-5](chapter03.assets/image-20241114195926180.png)

![3-6](chapter03.assets/image-20241114195934686.png)

注意：没有movzlq ，原因是movl自带movzlq的功能。

> #### 字节传送指令的比较
>
> ![image-20241114200326036](chapter03.assets/image-20241114200326036.png)

练习题3-2：请说出下面代码的错误

```assembly
movl %rax,(%rsp)         #指令后缀与寄存器ID不匹配
movw (%rax),4(%rsp)		 #不能同时将源和目标都设为内存引用。
movb %al,%sl   			#没有名为%sl的寄存器
movq %rax,$0x123		#不能以立即数作为目标操作数
movl %eax,%rdx			#目的操作数大小不正确
movb %si,8(%rbp)         #指令后缀与寄存器ID不匹配
```



### 3.4.3 数据传送示例

c代码如下：

```cpp
long exchange(long *xp,long y){
  long x = *xp;
  *xp = y;
  return x;

}
```

得到其汇编代码：

```shell
root@canppx00003:~/wfd/cpp_code# gcc -Og -c exchange.c
root@canppx00003:~/wfd/cpp_code# objdump -d exchange.o

exchange.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <exchange>:
   0:   f3 0f 1e fa             endbr64
   4:   48 8b 07                mov    (%rdi),%rax
   7:   48 89 37                mov    %rsi,(%rdi)
   a:   c3                      ret

```

通过上述代码，我们可以知道：

- exchange指令通过三条指令实现，两个数据传送，一条返回函数被调用点的指令（ret）
- 过程参数分别存储在 %rdi(第一个参数xp)和%rsi(第二个参数y)寄存器中。
- C语言所谓的指针其实就是地址。**间接引用指针就是将该指针存放在一个寄存器中，然后在内存引用中使用这个寄存器。**
- 像`x`这样的局部变量保存在寄存器中。

> **练习题3.4**：假设变量sp和dp被声明为类型：
>
> ```
> src_t *sp;
> dest_t *dp;
> ```
>
> 这里src_t和dest_t是用typedef声明的数据类型。我们想使用适当的数据传送指令来实现下面的操作
>
> ```
> *dp = (dest_t) *sp;
> ```
>
> 假设sp和dp的值分别存储在寄存器%rdi和%rsi中。对于表中的每个表项，给出实现指令数据传送的两条指令。其中第一条指令应该从内存中读数，做适当的转换，并设置寄存器%rax的适当部分。然后，第二条指令要把%rax的适当部分写到内存。在这两种情况中，寄存器的部分可以使%rax、%eax、%ax或%al，两者可以互不相同。
>
> | src_t         | dest_t        | 指令                                        | 注释                              |
> | ------------- | ------------- | ------------------------------------------- | --------------------------------- |
> | long          | long          | `movq (%rdi),%rax `<br>`movq %rax,(%rsi)`   | 读8个字节<br>存8个字节            |
> | char          | int           | `movsbl (%rdi),%eax`<br>`movl %eax,(%rsi)`  | 将char转换为int<br>存4个字节      |
> | char          | unsigned      | `movzbl(%rdi),%eax`<br/>`movb %eax,(%rsi)`  | 将char转换为int<br/>存4个字节     |
> | unsigned char | long          | `movzbl (%rdi),%eax`<br/>`movq %eax,(%rsi)` | 读一个字节并零扩展<br/>存8个字节  |
> | int           | char          | `movl (%rdi),%eax`<br/>`movb %al,(%rsi)`    | 读4个字节<br>存低位字节           |
> | unsigned      | unsigned char | `movl (%rdi),%eax`<br/>`movb %al,(%rsi)`    | 读4个字节<br>存低位字节           |
> | char          | short         | `movsbw (%rdi),%ax`<br/>`movw %ax,(%rsi)`   | 读一个字节并符号扩展<br>存2个字节 |

> #### 练习题3.5
>
> 已知信息如下：将一个原型位：
>
> ```cpp
> void decode1(long *xp,long *yp,long *zp);
> ```
>
> 的函数编译成汇编代码，得到如下代码：
>
> ```
> void decodel (long *xp, long *yp, long *zp) xp in %rdi, yp in %rsi，zp in %rdx
> decodel:
> movq    (%rdi), %r8
> movq    (%rsi), %rcx
> movq    (%rdx), %rax
> movq    %r8, (%rsi)
> movq    %rcx, (%rdx)
> movq    %rax, (%rdi)
> ret
> ```
>
> 参数xp、yp和zp分别存储在对应的寄存器红%rdi、％rsi 和%rdx中。
>
> 请写出等效于上面汇编代码的decodel的C代码。
>
> ```cpp
> void decode1(long *xp,long *yp,long *zp){
>     long x = *xp;
>     long y = *yp;
>     long z = *zp;
>     
>     *yp = x;
>     *zp = y;
>     *xp = z;
> }
> ```
>
> 

### 3.4.4 压入和弹出栈数据

最后两个数据传送操作即入栈和出栈指令，指令如下：

![3-8](chapter03.assets/image-20241116145428781.png)



> 将一个四字值压入栈中，首先要将栈指针减8，然后将值写到新的栈顶位置。因此pushq %rbp等价于：
>
> ```
> subq $8,%rsp 
> movq %rbp,(%rsp)
> ```
>
> 弹出一个四字的操作包括从栈顶位置读出数据，然后将栈指针加8.因此push %rax等价于：
>
> ```
> movq (%rsp),%rax
> addq $8,%rsp
> ```
>
> 

在x86-64中，程序栈存放在内存中的某个区域。如下图所示，栈向下增长，这样一来，栈顶元素的地址是所有栈中元素地址中最低的。` 栈指针%rsp保存着栈顶元素的地址。`

![3-9](chapter03.assets/image-20241116150220840.png)

## 3.5 算数和逻辑操作

下图列出了x86-64的一些整数和逻辑操作。大多数操作都被分成指令类，例如：ADD指令类。每种指令类都对应四种不同大小数据的指令。

![3-10](chapter03.assets/image-20241116151030212.png)

### 3.5.1 加载有效地址

加载有效地址指令leaq实际上是movq的变形。他的指令形式的从内存读取数据到寄存器。

语法：

```assembly
leaq S,D
```

- 其中源内存地址，D只能是寄存器。
- 主要作用是**将有效地址写入到目的操作数（寄存器）**。
- 可以简介的实现算数操作，例如`%rdx` 的值为x ，则下面的指令可以实现 5x + 7

```assembly
leaq 7(%rdx,%rdx,4),%rdx
```

看下面c程序：

```cpp
long scale(long x,long y,long z){
    long t = x+4*y+12*z;
    return t;
}
```

编译时，该函数的算数运算以三条leaq指令实现:

![p129](chapter03.assets/image-20241118110950693.png)



### 3.5.2 一元和二元操作

**一元操作指令**

| 指令  | 效果         | 描述 |
| ----- | ------------ | ---- |
| INC D | `D+1  –>  D` | 加1  |
| DEC D | `D-1  –>  D` | 减1  |
| NEG D | `-D  –>  D`  | 取负 |
| NOT D | `~D  –>  D`  | 取补 |

> 操作数D可以是寄存器也可以是一个内存地址。
>
> 例如：incq (%rsp)  会使栈顶的8字节元素+1  (注：%ras中存储的是栈顶指针。（%rsp）在内存中获取栈顶指针对应的内容。)

**二元操作**



|      指令       | 效果         | 描述 |
| :-------------: | ------------ | ---- |
| ADD       S, D  | `D+S  –>  D` | 加   |
| SUB        S, D | `D-S  –>  D` | 减   |
| IMUL      S, D  | `D*S  –>  D` | 乘   |
| XOR       S, D  | `D^S  –>  D` | 异或 |
| OR         S, D | `D|S  –>  D` | 或   |
|  AND      S, D  | `D&S  –>  D` | 与   |

> 二元操作指令需要注意以下几点：
>
> - 第二个操作数既是源又是目的。
> - S可以是立即数，寄存器或者内存位置。D可以是寄存器，内存位置。
> - 当D为内存地址是，处理器必须从内存读出值，执行操作，再把结果写回内存。

### 3.5.3 位移操作

|      指令      | 效果             | 描述            |
| :------------: | ---------------- | --------------- |
| SAL      k, D  | `D<<k  –>  D`    | 左移            |
| SHL       k, D | `D<<k  –>  D`    | 左移(等同于SAL) |
| SAR      k, D  | D >>~A~ k  –>  D | 算数右移        |
| SHR      k, D  | D >>~L~ k  –>  D | 逻辑右移        |



> - k可以是立即数，或者放在单字节寄存器`%cl`(只允许以这个特定寄存器为操作数)
>
> - 移位操作对w位长的数据进行操作，移位量由%cl寄存器的m位决定，高位会被忽略。其中2^m^ = w
>
>   例如：当%cl 为 0xFF时指令：`salb 会移动 7位，salw会移动15位，sall会移动31位，salq会移动63位。`.
>
> - 移位操作的目的操作数可以使一个寄存器或者一个内存。

### 3.5.5 特殊算数运算（乘法和除法）

当两个64位有符号数或无符号数相乘得到的乘机需要128位。

x86-64指令集对128位（16字）数的操作提供有限支持。intel把16字节的数称为`“八字(otc word)”`。

- 乘法

| 指令                | 效果                              | 描述         |
| ------------------- | --------------------------------- | ------------ |
| imulq             S | `S X R[%rax] —>  R[%rdx]:R[%rax]` | 有符号全乘法 |
| mulq             S  | `S X R[%rax] —>  R[%rdx]:R[%rax]` | 无符号全乘法 |

> 这两个乘法指令都要求其中一个源操作数在%rax寄存器中，同时结果的高位64位在%rdx ，低64位在%rax中。
>
> 下面这段c代码演示了其过程：
>
> ```cpp
> #include <inttypes.h>
> typedef unsigned __int128 uint128_t;
> 
> void store_uprod(uint128_t *dest,uint64_t x,uint64_t y){
>     *dest = x * (uint128_t)y;
> }
> ```
>
> GCC生成的汇编代码如下：
>
> ![p134](https://i-blog.csdnimg.cn/direct/7ae9c8beab0041cf88884c04536e5af4.png)
>
> 从上面可以看出，存储乘积需要两个movq指令：一个存储低8个字节，一个存储高8个字节。由于针对小端机器，所以高位字节存储在大地址。

- 除法

| 指令                | 效果                                                         | 描述                                                   |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| idivq             S | `R[%rdx]:R[%rax] mod S —>  R[%rdx]`<br>`R[%rdx]:R[%rax] ÷ S —>  R[%rax]` | **有符号除法**<br>余数存储在%rdx中<br>商存储在%rax中。 |
| clto                | `符号扩展(R[%rax]) —>  R[%rdx]:R[%rax]`                      | 将%rax转化为8字                                        |

下面这段c代码展示了除法的过程，他计算了两个64位有符号数的商和余数。

```cpp
void remdiv ( long x, long y, long *qp , long *rp){
        long q=x/y;
        long r=x%y;
        *qp=q;
        *rp=r;
} 
```

GCC汇编代码如下：

![p135](https://i-blog.csdnimg.cn/direct/7d79d3b587254db9b158c7f0e3260398.png)



> 上述代码中：
>
> - 必须首先把参数qp保存到另一个寄存器，因为除法操作要使用到参数寄存器%rdx。
> - 无符号数除法使用divq指令，通常寄存器%rdx会事先设置为0.




## 3.6 控制



### 3.6.1 条件码

除了整数寄存器，CPU还维护了一组单个位的**条件码（condition code）寄存器**，作用是描述最近算数或逻辑操作的属性。

可以使用这一组寄存器来执行条件分支指令。

最常用的条件码有：

- CF：进位标志。最近操作使最高位产生了进位则为1。（可以用来检查无符号操作的溢出）
- ZF：零标志。最近的操作得出的结果为0。
- SF：符号标志。最IN的操作得到的结果为负数。
- OF：溢出标志。最近的操作导致一个补码溢出。

> 比如：一条ADD指令完成等待与C表达式t=a+b。则根据以下C表达式来设置条件码：
>
> ![p136](chapter03.assets/image-20241118122549811.png)
>
> ```
> (a<0 == b<0) && (t<0 !=a<0)
> ```
>
> 表达式含义是：a和b同号，t和a不同号 结果才为true

可以改变条件码的指令有：**算数指令，逻辑指令 、CMP指令和TEST指令。**

> 注意：leaq指令不改变任何条件码，因为他是用来进行地址计算的。

CMP指令和TEST指令只设置条件码，而不改变其他寄存器的值。

![3-13](chapter03.assets/image-20241118142111906.png)

> 例如：testq %rax，%rax用来检查%rax是负数，零还是正数。

### 3.6.2 访问条件码

条件码不能直接读取，常用的使用方法有三条：

1. 可以根据条件码的某种组合将一个字节设置为0或1
2. 可以条件跳转到程序的某个其他部分。
3. 可以有条件地传送数据。

对于第一种情况所对应的一整类指令称为SET指令

![3-14](chapter03.assets/image-20241119092429352.png)

> 关于set指令需要注意：
>
> - 目的操作数D是低位单字节寄存器之一，或是一个字节的内存位置。指令会将这个字节设置为0或1。此时为了得到一个32位或者64位结果，必须将高位清零。
> - 一个计算C语言表达式a<b的典型指令序列如下所示：
>
> ![p137](chapter03.assets/image-20241119092652733.png)

如何理解上述set指令的含义：

1. sete 即“当相等时设置（set when equal）”:当a=b时，a-b = 0 因此零标志位就表示相等
2. setl 即：“当小于时设置” 
   - 当没有发生溢出时，OF设置为0就表示无溢出，a-b<0 时，a<b 故SF设置为1. 当a-b>=0时，SF设置为0.
   - 当有溢出发生时，OF设置为1，此时 a-b>0（负溢出） 时a<b ；而当a-b<0（正溢出时）a>b。故当OF被设置为1时，当且仅当SF被设置为0，有a<b
3. 对于无符号比较测试，现在a和b时变量a和b的无符号形式表示的整数。在执行t = a-b时，当a-b<0，CMP指令会设置进位标志。因而无符号比较使用的是进位标志和零标志的组合。



### 3.6.3 跳转指令

跳转指令-JUMP会导致执行切换到程序中一个全新的位置。

跳转目的通常用一个标号（label）指明。在产生目标文件时，汇编器会确定所有带标号指令的地址，并将（跳转目标）编码为跳转指令的一部分。

- 直接跳转：跳转的目标作为指令的一部分编码，在汇编中，直接跳转时给出一个标号作为跳转目标。

```
jump .L1
```



- 间接跳转：跳转目标从寄存器或者内存中读取。

```
jump *%rax		//以寄存器%rax中的值作为跳转目标
jump *(%rax)	//以%rax中的值作为读地址，从内存中读出跳转目标
```

![3-15](chapter03.assets/image-20241119112551511.png)

> 上述表中跳转指令都是有条件地，他会根据条件码的某种组合完成跳转。

### 3.6.4 跳转指令编码

理解跳转指令的目标如何编码对理解链接过程非常重要。

在汇编代码中，跳转目标用符号编号书写，汇编器以及后来的连接器会产生跳转目标适当的编码。

跳转指令有几种不同的编码：

- PC相对的编码（最常用）：他会将目标指令的地址与紧跟在跳转指令后面那条指令的地址之间的差作为编码。地址的偏移量可以编码为1,2，或4个字节。
- 绝对地址：用4个字节直接指定编码。

下面是一个PC相对寻址的例子：

![p140](chapter03.assets/image-20241119113023270.png)

反汇编格式如下：

![p140](chapter03.assets/image-20241119113036512.png)

> 第2行跳转指令的跳转目标指明为 0x8 .第五行跳转目标是0x5.观察上述编码 第一条跳转指令的跳转目标为0x3 + 下一条的指令（第3行）指令地址 0x5 = 0x8
>
> 第二条跳转指令 的跳转目标为0x5  = 0xf8 (10进制-8)+下一条指令的指令地址：0xd  。  即第3行的指令的地址。

下面是链接后的反汇编版本：

![p140](chapter03.assets/image-20241119113610723.png)

其跳转地址和上述代码是相同的。

### 3.6.5 用条件控制来实现条件分支

![3-16](chapter03.assets/image-20241119113719268.png)

C语言中if-else语句的通用形式：

```cpp
if(test-expr)
    then-statement
else
    else-statement
        
```

汇编通常用下面的形式：

```
t = test-expr;
if(!t)
   goto false;
then-statement
goto done;
false:
   else-statement;
done:
```

> 练习题：根据下面的汇编代码补充C代码：
>
> ```assembly
> root@canppx00003:~/wfd/cpp_code/chapter03# objdump -d demo_3_18.o
> 
> demo_3_18.o:     file format elf64-x86-64
> 
> 
> Disassembly of section .text:
> 
> 0000000000000000 <test>:
>    0:   f3 0f 1e fa             endbr64
>    4:   48 8d 04 37             lea    (%rdi,%rsi,1),%rax
>    8:   48 01 d0                add    %rdx,%rax
>    b:   48 83 ff fd             cmp    $0xfffffffffffffffd,%rdi
>    f:   7d 15                   jge    26 <test+0x26>
>   11:   48 39 d6                cmp    %rdx,%rsi
>   14:   7d 08                   jge    1e <test+0x1e>
>   16:   48 89 f8                mov    %rdi,%rax
>   19:   48 0f af c6             imul   %rsi,%rax
>   1d:   c3                      ret
>   1e:   48 89 f0                mov    %rsi,%rax
>   21:   48 0f af c2             imul   %rdx,%rax
>   25:   c3                      ret
>   26:   48 83 ff 02             cmp    $0x2,%rdi
>   2a:   7e 07                   jle    33 <test+0x33>
>   2c:   48 89 f8                mov    %rdi,%rax
>   2f:   48 0f af c2             imul   %rdx,%rax
>   33:   c3                      ret
> 
> ```
>
> c代码如下：
>
> ```cpp
> 
> long test(long x,long y,long z){
>     long val = x+y+z;
>     if(x<-3){
>         if(y<z)
>             val = x*y;
>         else
>             val = y*z;
> 
>     }else if(x>2){
>         val = x*z;
>     }
>     return val;
> }
> ```
>
> 



### 3.6.6用条件传送实现分支条件

实现条件操作的传统方法是使用控制的条件转移。一种替代策略是使用数据的条件转移。条件传送使用场景受限，但是他更符合现代处理器的性能特性。

下面的是使用条件传送的代码示例：

![3.17](chapter03.assets/image-20241120104257363.png)



> 为什么条件数据传送比条件控制转移的代码性能好？
>
> 由于处理器是使用流水线来获取高性能。在一个时钟周期处理一条指令的一小部分。当使用条件控制时，如果条件不满足时。后续加载到cpu中的指令有一部分会被跳过，跳过的部分会浪费一分部时钟周期（15~30个）。

条件传送会使用到cmove指令，cmov指令的结构和含义如下：

![3.18](chapter03.assets/image-20241120104627501.png)

使用条件传送指令处理器无需预测测试结果就可以执行传送代码。

考虑下面的条件表达式和复制的通用形式：

```c
v = test-expr ? then-expr : else-expr
```

用条件控制转移的标准方式来编译这个表达式，结果如下：

```c
if(~test-expr)
	goto false;
v = then-expr;
goto done;
false:
	v = else-expr;
done:
```

使用条件传送来编译，形式如下：

```c
v = then-expr;
ve = else-expr;
t = test-expr;
if(!t) v = ve;
```

不是所有的表达式都可以用条件传送来编译。then-expr和else-expr都求值，且表达式中的任意一个可能产生错误条件或者副作用，就会导致非法行为。例如下面的例子：

```cpp
long cread(long *xp){
    return (xp ? *xp:0);
}
```

使用条件传送汇编代码如下：

![p148](chapter03.assets/image-20241120105304475.png)

> 上述实现是非法的，因为测试位假时，即xp是空指针，但是movq指令对xp的间接引用还是发生了，会导致一个间接引用为空指针的错误。所以必须使用分支代码来编译这段代码。

总的来说，条件数据传送提供了一种用条件控制转移来实现条件操作的替代策略。他只能用于非常受限的情况。而且与现代处理器的运行方式更契合。



## 3.6.7 循环

#### 1.do-while循环

通用形式如下：

```cpp
do
	body-statement
	while(test-expr)
```

这种形式可以被翻译成如下所示的条件和goto语句：

```cpp
loop:
	body-statement
	t = test-expr;
	if(t)
		goto loop;
```

![3-19](chapter03.assets/image-20241120160652108.png)

#### 2.while循环

通用形式如下：

```cpp
while(test-expr)
	body-statement
```

Gcc在代码生成中使用两种方式翻译while循环。

- 方式1：跳转到中间，执行一个无条件跳转到循环结尾的测试，以此来执行初始的测试。

```cpp
goto test;
loop:
	body-statement;
test:
	t = test-expr;
	if(t)
		goto loop;
```

![3-20](chapter03.assets/image-20241120161101625.png)

- 方式2：称之为guarded-do，受限采用条件分支，如果初始条件不成立就跳过循环，把代码编程do-while循环。 当使用较高优化登记编译时，例如使用选项 -O1，Gcc会采用这种策略。

可以翻译为do-while

```cpp
t = test-expr;
if(!t)
	goto done;
do
	body-statement
	while(test-expr);
done;
```

也可以翻译为goto格式：

```cpp
t = test-expr;
if(!t)
	goto done;
loop:
	body-statement
	t = test-expr;
	if(t)
		goto loop;
done;
```



#### 3.for循环

for循环的通用定时如下：

```cpp
for(init-expr;test-expr;update-expr)
    body-statement
```

可以使用while循环转换:

```cpp
init-expr;
while(test-expr){
    body-statement
    update-expr;
}
```

GCC为for循环产生的代码是while循环的两个中翻译之一，这取决于优化登记。

跳转到中间策略会得到如下goto代码：

```
init-expr
goto test;
loop:
	body-statement;
	update-expr
test:
	t = test-expr;
	if(t)
		goto loop;
```

而guarded-do策略得到：

```cpp
init-expr
t = test-expr;
if(!t)
	goto done;
do
	body-statement
    update-expr
    t=test-expr;
	if(t)
        goto loop;
done;
```

