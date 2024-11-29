
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
