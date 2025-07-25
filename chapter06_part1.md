# 第六章 存储器层次结构

实际上，存储器系统（memorysystem）是一个具有不同容量、成本和访问时间的存储设备的层次结构。

- CPU寄存器保存着最常用的数据。
- 靠近CPU的小的、快速的高速缓存存储器（cachememory）作为一部分存储在相对慢速的主存储器（mainmemory）中数据和指令的缓冲区域。
- 主存缓存存储在容量较大的、慢速磁盘上的数据，
- 而这些磁盘常常又作为存储在通过网络连接的其他机器的磁盘或磁带上的数据的缓冲区域。

## 6.1 存储级数

### 6.1.1 随机访问存储器



随机访问存储器（Random-AccessMemory，RAM)分为两类：静态的和动态的。静态 RAM(SRAM）比动态RAM（DRAM)更快，但也贵得多。

- SRAM用来作为高速缓存存储器，既可以在CPU芯片上，也可以在片下。
- DRAM用来作为主存以及图形系统的顿缓冲区。

>  典型地，一个桌面系统的SRAM不会超过几兆字节，但是DRAM却有儿百或几千兆字节。

#### 1.静态RAM

SRAM将每个位存储在一个双稳态的（bistable）存储器单元里。每个单元是用一个六晶体管电路来实现的。这个电路有这样一个属性，**<u>*它可以无限期地保持在两个不同的电压配置（configuration）或状态（state）之一。其他任何状态都是不稳定的一一从不稳定状态开始，电路会迅速地转移到两个稳定状态中的一个*</u>**。这样一个存储器单元类似于图6-1中画出的倒转的钟摆。

![图6-1中](assert/image-20250623111434042.png)

由于SRAM存储器单元的双稳态特性，只要有电，它就会永远地保持它的值。即使有干扰（例如电子噪音）来扰乱电压，当干扰消除时，电路就会恢复到稳定值。

#### 2.动态RAM

DRAM将每个位存储为对一个电容的充电。这个电容非常小，通常只有大约30毫微微法拉（femtofarad）30×10-15法拉。DRAM存储器可以制造得非常密集一一每个单元由一个电容和一个访问晶体管组成。但是，与SRAM不同，DRAM存储器单元对干扰非常敏感。当电容的电压被扰乱之后，它就永远不会恢复了。暴露在光线下会导致电容电压改变。实际上，数码照相机和摄像机中的传感器本质上就是DRAM单元的阵列。

很多原因会导致漏电，使得DRAM单元在10～100毫秒时间内失去电荷。**幸运的是计算机运行的时钟周期是以纳秒来衡量的，所以相对而言这个保持时间是比较长的。内存系统必须周期性地通过读出，然后重写来刷新内存每一位。**有些系统也使用纠错码，其中计算机的字会被多编码几个位（例如64位的字可能用72位来编码），这样一来，电路可以发现并纠正一个字中任何单个的错误位。



#### 3.传统的DRAM

DRAM芯片中的单元（位）被分成d个超单元（supercell），每个超单元都由w个DRAM 单元组成。

一个d x w 的DRAM总共存储了d·w位信息。超单元被组织成一个r行c列的长方形阵列，这里rc=d。每个超单元有形如（i，j)的地址，这里i表示行，而j表示列。

例如，图6-3展示的是一个16×8的DRAM芯片的组织，有d=16个超单元，每个超单元有二8位，r=4行，c=4列。带阴影的方框表示地址（2，1）处的超单元。信息通过称为引脚（pin）的外部连接器流人和流出芯片。每个引脚携带一个1位的信号。

![图6-3](assert/image-20250623113006524.png)

每个DRAM芯片被连接到某个称为内存控制器（memorycontroller）的电路，这个电路可以一次传送w位到每个DRAM芯片或一次从每个DRAM芯片传出w位

例如，要从图6-3中16×8的DRAM中读出超单元（2，1）

- 内存控制器发送行地址 2，如图6-4a所示。DRAM的响应是将行2的整个内容都复制到一个内部行缓冲区。
- 接下来，内存控制器发送列地址1，如图6-4b所示。DRAM的响应是从行缓冲区复制出超单元（2，1）中的8位，并把它们发送到内存控制器。

![6-4](assert/image-20250623113820859.png)

DRAM组织成二维阵列相比于线性结构可以减少芯片上地址引脚的数量。但是分布发送地址增加了访问时间。

#### 4.内存模块

DRAM芯片封装在内存模块（memory module）中，它插到主板的扩展槽上。Corei7 系统使用的240个引脚的双列直插内存模块（Dual Inline Memory Module，DIMM），它以 **64位为块**传送数据到内存控制器和从内存控制器传出数据。

图6-5展示了一个内存模块的基本思想。示例模块用8个8M×8（64Mbit）的DRAM芯片，总共存储64MB（兆字节），这8个芯片编号为0～7。每个超单元存储主存的一个字节用相应超单元地址为（i,j）的8个超单元来表示主存中字节地址A处的64位字。在图6-5 的示例中，DRAM0存储第一个（低位)字节，DRAM1存储下一个字节，依此类推。

![图6-5](assert/image-20250623114727198.png)

#### 5.增强的DRAM

下列每种都是基于传统的DRAM单元，并进行一些优化，提高访问基本 DRAM单元的速度。

- 快页模式DRAM(Fast Page Mode DRAM，FPM DRAM)：传统的DRAM将超单元的一整行复制到它的内部行缓冲区中，使用一个，然后丢弃剩余的。FPM DRAM允许对同一行连续地访间可以直接从行缓冲区得到服务，从而改进了这一点。例如：要从一个FPM DRAM的同一行中读取4超单元，内存控制器发送第一个RAS/CAS请求，后面跟三个CAS请求。

- 扩展数据输出DRAM(Extended Data Out DRAM，EDO DRAM)：FPM DRAM的一个增强的形式，它允许各个CAS信号在时间上靠得更紧密一点。

- 同步DRAM(Synchronous DRAM，SDRAM）。就它们与内存控制器通信使用一组显式的控制信号来说，常规的、FPM和EDODRAM都是异步的。SDRAM用与驱动内存控制器相同的外部时钟信号的上升沿来代替许多这样的控制信号。我们不会深入讨论细节最终效果就是SDRAM能够比那些异步的存储器更快地输出它的超单元的内容。

- 双倍数据速率同步DRAM（Double Data-Rate Synchronous DRAM，DDRS DRAM)。 DDRSD RAM是对SDRAM的一种增强，它通过使用两个时钟沿作为控制信号从而使DRAM的速度翻倍。不同类型的DDR SDRAM是用提高有效带宽的很小的预取缓冲区的大小来划分的：DDR（2位）、DDR2（4位）和DDR3（8位）。

#### 6.非易失存储器

非易失性存储器（nonvolatile memory）即使是在关电后，仍然保存着它们的信息。

现在有很多种非易失性存储器。由于历史原因，虽然ROM中有的类型既可以读也可以写，但是它们整体上都被称为只读存储器（Read-OnlyMemory，ROM）。 ROM是以它们能够被重编程（写）的次数和对它们进行重编程所用的机制来区分的。

- PROM(Programmable ROM，可编程ROM）只能被编程一次。PROM的每个存储器单元有一种熔丝（fuse），只能用高电流熔断一次。

- 可擦写可编程ROM（Erasable Programmable ROM，EPROM）有一个透明的石英窗口，允许光到达存储单元。紫外线光照射过窗口，EPROM单元就被清除为0。对EPROM编程是通过使用一种把1写人EPROM的特殊设备来完成的。EPROM能够被擦除和重编程的次数的数量级可以达到1000次。
  - 电子可擦除PROM（Electrically Erasable PROM，EEPROM类似于EPROM，但是它不需要一个物理上独立的编程设备，因此可以直接在印制电路卡上编程。EEPROM能够被编程的次数的数量级可以达到10^5^次。
- 闪存（flashmemory）是一类非易失性存储器，基于EEPROM，它已经成为了一种重要的存储技术。

存储在ROM设备中的程序通常被称为固件（firmware）。当一个计算机系统通电以后它会运行存储在ROM中的固件。一些系统在固件中提供了少量基本的输入和输出函数一例如PC的BIOS（基本输入/输出系统）例程。

#### 7.访问主存

数据流通过称为总线（bus）的共享电子电路在处理器和DRAM主存之间来来回回。

每次CPU和主存之间的数据传送都是通过一系列步骤来完成的，这些步骤称为总线事务（bus transaction）。

- 读事务（read transaction）从主存传送数据到CPU。
- 写事务（write trans-action）从CPU传送数据到主存。

![6-5](assert/image-20250623155012550.png)

I/O桥接器将系统总线的电子信号翻译成内存总线的电子信号。

考虑当CPU执行一个如下加载操作时会发生什么？

```c
 movq A,%rax
```

<img src="assert/image-20250623155343025.png" alt="image-20250623155343025" style="zoom:50%;" />

反过来，当CPU执行一个像下面这样的存储操作时 

```c
movq %rax,A
```

寄存器%rax的内容被写到地址A，CPU发起写事务。同样，有三个基本步骤。

- 首先 CPU将地址放到系统总线上。内存从内存总线读出地址，并等待数据到达（图6-8a）。
- 接下来，CPU将%rax中的数据字复制到系统总线（图6-8b）。
- 最后，主存从内存总线读出数据字，并且将这些位存储到DRAM中（图6-8c）。

<img src="assert/image-20250623155602291.png" alt="image-20250623155602291" style="zoom: 67%;" />
## 6.2 局部性

一个编写良好的计算机程序常常具有良好的局部性（locality）。也就是，它们倾向于引用邻近于其他最近引用过的数据项的数据项，或者最近引用过的数据项本身。这种倾向性，被称为局部性原理（principleoflocality），是个持久的概念，对硬件和软件系统的设计和性能都有着极大的影响。

局部性通常有两种不同的形式：时间局部性（temporallocality）和空间局部性（spatia）locality）。在一个具有良好时间局部性的程序中，被引用过一次的内存位置很可能在不远的将来再被多次引用。在一个具有良好空间局部性的程序中，如果一个内存位置被引用了一次，那么程序很可能在不远的将来引用附近的一个内存位置。

### 6.2.1 对程序数据引用的局部性





## 6.3 存储器层次结构

6.1节和6.2节描述了存储技术和计算机软件的一些基本的和持久的属性：

- 存储技术：不同存储技术的访问时间差异很大。速度较快的技术每字节的成本要比速度较慢的技术高，而且容量较小。CPU和主存之间的速度差距在增大。
- 计算机软件：一个编写良好的程序倾向于展示出良好的局部性。

<img src="assert/image-20250623160520880.png" alt="6-21" style="zoom:67%;" />

### 6.3.1 存储器层次结构中的缓存

一般而言，高速缓存（cache，读作“cash）是一个小而快速的存储设备，它作为存储在更大、也更慢的设备中的数据对象的缓冲区域。使用高速缓存的过程称为缓存（caching：读作“cashing）。

存储器层次结构的中心思想是，对于每个K，位于K层的更快更小的存储设备作为位于K十1层的更大更慢的存储设备的缓存。

图6-22展示了存储器层次结构中缓存的一般性概念。

- 第K+1层的存储器被划分成连续的数据对象组块（chunk），称为块（block）。
- 每个块都有一个唯一的地址或名字，使之区别于其他的块。
- 块可以是固定大小的（通常是这样的），也可以是可变大小的（例如存储在 Web服务器上的远程HTML文件）。例如，图6-22中第k十1层存储器被划分成16个大小固定的块，编号为0～15。
- 数据以块为大小传输单元在层与层之间复制。虽然在层次结构中任何一对相邻的层次之间块大小是固定的，但是其他的层次对之间可以有不同的块大小。一般而言，层次结构中较低层（离CPU较远）的设备的访问时间较长，因此为了补偿这些较长的访问时间，倾向于使用较大的块。

![6-22](assert/image-20250623164324954.png)

类似地，第k层的存储器被划分成较少的块的集合：每个块的大小与k十1层的块的大小一样。在任何时刻，第k层的缓存包含第k十1层块的一个子集的副本。

#### 1.缓存命中

当程序需要第尺十1层的某个数据对象d时，它首先在当前存储在第尺层的一个块中查找d。如果d刚好缓存在第k层中，那么就是我们所说的缓存命中（cache hit）。

#### 2.缓存不命中

另一方面，如果第尺层中没有缓存数据对象d，那么就是我们所说的缓存不命中（cache miss）。当发生缓存不命中时，第k层的缓存从第k十1层缓存中取出包含d的那个块，如果第尺层的缓存已经满了，可能就会覆盖现存的一个块。

覆盖一个现存的块的过程称为替换（replacing）或驱逐（evicting）这个块。被驱逐的这个块有时也称为牺牲块（victim block）。决定该替换哪个块是由缓存的替换策略（replacement policy）来控制的。例如，一个具有随机替换策略的缓存会随机选择一个牺牲块。

#### 3.缓存不命中的种类

一个空的缓存有时被称为冷缓存（cold cache），此类不命中称为强制性不命中（compulsory miss）或冷不命中（cold miss）。冷不命中很重要，因为它们通常是短暂的事件，不会在反复访问存储器使得缓存暖身（warmed up）之后的稳定状态中出现

只要发生了不命中，第k层的缓存就必须执行某个`放置策略（placement policy）`，确定把它从第k+1层中取出的块放在哪里。最灵活的替换策略是允许来自第k+1层的任何块放在第k层的任何块中。对于存储器层次结构中高层的缓存（靠近CPU），它们是用硬件来实现的，而且速度是最优的，这个策略实现起来通常很昂贵，因为随机地放置块，定位起来代价很高。

因此，硬件缓存通常使用的是更严格的放置策略，这个策略将第K+1层的某个块限制放置在第K层块的一个小的子集中（有时只是一个块）。

> 例如，在图6-22中，我们可以确定第k十1层的块i必须放置在第k层的块（i mod 4）中。例如，第k十1层的块0、4、8 和12会映射到第k层的块0；块1、5、9和13会映射到块1；依此类推。注意，图6-22 中的示例缓存使用的就是这个策略。
>
> 

这种限制性的放置策略会引起一种不命中，称为***冲突不命中（conflict miss）***，在这种情况中，缓存足够大，能够保存被引用的数据对象，但是因为这些对象会映射到同一个缓存块，缓存会一直不命中。例如，在图6-22中，如果程序请求块0，然后块8，然后块0，然后块8，依此类推，在第k层的缓存中，对这两个块的每次引用都会不命中，即使这个缓存总共可以容纳4个块。

程序通常是按照一系列阶段（如循环）来运行的，每个阶段访问缓存块的某个相对稳定不变的集合。例如，一个嵌套的循环可能会反复地访问同一个数组的元素。这个块的集合称为这个阶段的工作集（working set）。当工作集的大小超过缓存的大小时，缓存会经历容量不命中（capacity miss）。换句话说就是，缓存太小了，不能处理这个工作集。

总结：

缓存不命中的种类：

- 强制性不命中（compulsory miss）或冷不命中（cold miss）：缓存是空的造成
- 冲突不命中（conflict miss）：缓存足够大，能够保存被引用的数据对象，但是因为这些被引用的对象会映射到同一个缓存块，缓存会一直不命中。
- 容量不命中（capacity miss）：缓存太小了，不能处理这个工作集。

#### 4.缓存管理

在每一层上，某种形式的逻辑必须管理缓存。这里，我们的意思是指某个东西要将缓存划分成块，在不同的层之间传送块，判定是命中还是不命中，并处理它们。管理缓存的逻辑可以是硬件、软件，或是两者的结合。

- 编译器管理寄存器文件，缓存层次结构的最高层。它决定当发生不命中时何时发射加载，以及确定哪个寄存器来存放数据。
- L1、L2和L3层的缓存完全是由内置在缓存中的硬件逻辑来管理的。
- 在一个有虚拟内存的系统中，DRAM主存作为存储在磁盘上的数据块的缓存，是由操作系统软件和CPU上的地址翻译硬件共同管理的。
- 对于一个具有像AFS这样的分布式文件系统的机器来说，本地磁盘作为缓存，它是由运行在本地机器上的AFS客户端进程管理的。

在大多数时候，缓存都是自动运行的，不需要程序采取特殊的或显式的行动。

![6-23](assert/image-20250623170236092.png)


## 6.4 高速缓存存储器

早期计算机系统的存储器层次结构只有三层：CPU寄存器、DRAM主存储器和磁盘存储。不过，由于CPU和主存之间逐渐增大的差距，系统设计者被迫在CPU寄存器文件和主存之间插人厂一个小的SRAM高速缓存存储器，称为L1高速缓存（一级缓存），如图6-24所示。L1高速缓存的访问速度儿乎和寄存器一样快，典型地是大约4个时钟周期

![6-24](assert/image-20250623171546101.png)

随着CPU和主存之间的性能差距不断增大，系统设计者在L1高速缓存和主存之间又插入了一个更大的高速缓存，称为L2高速缓存，可以在大约10个时钟周期内访问到它。有些现代系统还包括有一个更大的高速缓存，称为L3高速缓存，在存储器层次结构中，它位于L2高速缓存和主存之间，可以在大约50个周期内访问到它。虽然安排上有相当多的变化，但是通用原则是一样的。

### 6.4.1 通用高速缓存存储器

考虑一个计算机系统，其中每个存储器地址有m位，形成M=2^m^个不同的地址。如图6-25a所示，这样一个机器的高速缓存被组织成一个有S=2^s^个高速缓存组（cacheset）的数组。每个组包含E个高速缓存行（cacheline）。每个行是由个B=2^b^字节的数据块（block）组成的，一个有效位（validbit）指明这个行是否包含有意义的信息，还有t=m-（b+s）个标记位（tagbit）（是当前块的内存地址的位的一个子集），它们唯一地标识存储在这个高速缓存行中的块。

<img src="assert/image-20250624090100546.png" alt="image-20250624090100546" style="zoom:67%;" />

一般而言，高速缓存的结构可以用元组（S，E，B，m）来描述。高速缓存的大小（或容量）C指的是所有块的大小的和。标记位和有效位不包括在内。因此，C=S×E x B。

- S:高速缓存被组织成一个有S=2^s^个高速缓存组（cacheset）的数组。
- E:每个组包含E个高速缓存行（cacheline）。
- B:每个行是由个B=2^b^字节的数据块（block）组成的
- m:一个计算机系统，其中每个存储器地址有m位，形成M=2^m^个不同的地址。

当-条加载指令指示CPU从主存地址A中读一个字时，它将地址A发送到高速缓存。如果高速缓存正保存着地址A处那个字的副本，它就立即将那个字发回给CPU。**<u>*那么高速缓存如何知道它是否包含地址A处那个字的副本的呢？*</u>**高速缓存的结构使得它能通过简单地检查地址位，找到所请求的字，类似于使用极其简单的哈希函数的哈希表。下面介绍它是如何工作的：

缓存地址在组相联映射（Set-Associative Mapping）中的解析方式：参数S和B将m个地址位分为了三个字段，如图6-25b所示。

在组相联缓存中，**地址A被划分为3个字段**：

1. **块偏移位（B位）**：确定字在数据块中的具体位置；
2. **组索引位（S位）**：确定该地址对应缓存中的哪一个组；
3. **标记位（T位）**：用于在指定的组中识别是否命中某一行。

命中条件：要判断某一行是否命中，**必须满足两个条件**：

1. 该行的**有效位已设置**；
2. **地址中的标记位** 与 **该行存储的标记** 完全一致。

![6-26](assert/image-20250624091237890.png)

### 6.4.2 直接映射高速缓存

根据每个组的高速缓存行数E，高速缓存被分为不同的类。每个组只有一行（E一1）的高速缓存称为直接映射高速缓存（direct-mappedcache）（见图6-27）。

![6-27](assert/image-20250624092227386.png)

高速缓存确定一个请求是否命中，然后抽取出被请求的字的过程，分为三步：1）组选择；2 行匹配；3）字抽取。

#### 1.直接映射高速缓存中的组选择

![6-28](assert/image-20250624104005285.png)

#### 2.直接映射高速缓存中的行匹配和字选择

![6-29](assert/image-20250624104042782.png)

#### 3.直接映射高速缓存中不命中时的行替换

如果缓存不命中，那么它需要从存储器层次结构中的下一层取出被请求的块，然后将新的块存储在组索引位指示的组中的一个高速缓存行中。一般而言，如果组中都是有效高速缓存行了，那么必须要驱逐出一个现存的行。对于直接映射高速缓存来说，每个组只包含有一行，替换策略非常简单：用新取出的行替换当前的行。

#### 4.直接映射高速缓存中的冲突不命中

当程序访问大小为 2的幂的数组时，直接映射高速缓存中通常会发生冲突不命中。例如，考虑一个计算两个向量点积的函数：

```c
float dotprod(float x[8],float y[8])
{
    float sum = 0;
    int i = 0;
    for(i = 0;i<8;i++){
        sum += x[i] * y[i];
    }
    return sum;
}
```

对于x和V来说，这个函数有良好的空间局部性，因此我们期望它的命中率会比较高。不幸的是，并不总是如此。

假设浮点数是4个字节，x被加载到从地址0开始的32字节连续内存中，而y紧跟在 x之后，从地址32开始。为了简便，假设一个块是16个字节（足够容纳4个浮点数），高速缓存由两个组组成，高速缓存的整个大小为32字节。我们会假设变量sum实际上存放在一个CPU寄存器中，因此不需要内存引用。根据这些假设每个x[i]和y[i]会映射到相同的高速缓存组：

![image-20250624105016291](assert/image-20250624105016291.png)

在运行时，循环的第一次迭代引用x[0]，缓存不命中会导致包含x[0]～x[3]的块被加载到组0。接下来是对y[0]的引用，又一次缓存不命中，导致包含y[0]～y[3]的块被复制到组0，覆盖前一次引用复制进来的x的值。在下一次迭代中，对x[1]的引用不命中，导致x[0]～x[3]的块被加载回组0，覆盖掉y[0]～y[3]的块。因而现在我们就有了一个冲突不命中，而且实际上后面每次对x和V的引用都会导致冲突不命中，因为我们在 x和y的块之间抖动（thrash）。术语“抖动”描述的是这样--种情况，即高速缓存反复地加载和驱逐相同的高速缓存块的组。

简要来说就是，即使程序有良好的空间局部性，而且我们的高速缓存中也有足够的空间来存放x[i]和y[i]的块，每次引用还是会导致冲突不命中，这是因为这些块被映射到了同一个高速缓存组。这种抖动导致速度下降2或3倍并不稀奇。另外，还要注意虽然我们的示例极其简单，但是对于更大、更现实的直接映射高速缓存来说，这个问题也是很真实的。

幸运的是，一旦程序员意识到了正在发生什么，就很容易修正抖动问题。个很简单的方法是在每个数组的结尾放B字节的填充。例如，不是将x定义为floatx[8]，而是定义成 f1oatx[12]。假设在内存中y紧跟在x后面，我们有下面这样的从数组元素到组的映射：

![image-20250624110227528](assert/image-20250624110227528.png)

> #### 旁注：为什么用中间为位来做索引
>
> 如果高位用做索引，那么一些连续的内存块就会映射到相同的高速缓存块。
>
> 相比较而言，以中间位作为索引，相邻的块总是映射到不同的高速缓存行。
>
> ![6-31](assert/image-20250624110522820.png)

练习题 6.11 假象一个高速缓存，用地址的高s位做组索引，那么内存块连续的片会被映射到同一个高速缓存组。

A.每个这样的连续的数组片中有多少个块？
B.考虑下面的代码，它运行在-个高速缓存形式为（S，E，B，m）=（512，1，32，32）的系统上：

```c
int array[4096];
for( int i = 0; i< 4096; i++)
    sum += array[i]
```

在任意时刻，存储在高速缓存中的数组块的最大数量为多少？

> 解答：
>
> A: 用高位做索引，每个连续的数组片 有 2^t^个快组成，这里t时标记位数。因此，数组头2^t^个连续的快都会被映射到数组0，接下来的2^t^个快会被映射到组1.依次类推。
>
> B:对于直接映射高速缓存（S，E，B，m）=（512，1，32，32）高速缓存容量是512个32字节的块，每个高速缓存行中有t=18个标记位。因此，数组中头2^18^个块会映射到组0，接下来2^18^个块会映射到组1。因为我们的数组只由（4096×4）/32=512个块组成，所以数组中所有的块都被映射到组0。因此：在任何时刻，高速缓存至多只能保存一个数组块，即使数组足够小，能够完全放到高速缓存中。很明显，用高位做索引不能充分利用高速缓存。
>
> ![image-20250624115353458](assert/image-20250624115353458.png)

### 6.4.3 组相连高速缓存

直接映射高速缓存中冲突不命中造成的问题源于每个组只有一行（或者，按照我们的术语来描述就是E=1）这个限制。（每个组只有一个块）

组相联高速缓存（setassociativecache）放松了这条限制，所以每个组都保存有多于一个的高速缓存行。一个1<E<C/B的高速缓存通常称为E路组相联高速缓存。在下节中，我们会讨论E=C/B这种特殊情况（即只有一个组）。

![6-32](assert/image-20250624115654477.png)

#### 1.组选择

![6-33](assert/image-20250624115744251.png)

#### 2.行匹配和字选择

![6-34](assert/image-20250624115807177.png)

#### 3.缓存不命中时的行替换

如果CPU请求的字不在组的任何一行中，那么就是缓存不命中，高速缓存必须从内存中取出包含这个字的块。不过，一旦高速缓存取出了这个块，该替换哪个行呢？当然，如果有一个空行，那它就是个很好的候选。但是如果该组中没有空行，那么我们必须从中选择一个非空的行，希望CPU不会很快引用这个被替换的行。

✅替换过程简述：

1. **定位组**：根据地址中的组索引位，定位目标组。
2. **检查是否命中**：
   - 若组中某行的有效位为 1 且标签匹配 → 命中。
   - 若没有命中 → 不命中。
3. **查找空行**：
   - 若组中有行的有效位为 0 → 直接填入新块，无需替换。
4. **替换行**（所有行都有效）：
   - 需要根据**替换策略（replacement policy）**选择一行被替换。

🔁 常见替换策略：

| 策略                    | 说明                                 |
| ----------------------- | ------------------------------------ |
| **LRU**（最近最少使用） | 替换最近最少被访问的那一行（最常见） |
| **FIFO**（先进先出）    | 替换最早被加载进来的那一行           |
| **Random**（随机）      | 随机选择一行替换（硬件简单）         |
| **MRU**（最近最常用）   | 替换最近被使用的那一行（少见）       |

> 多数现代 CPU 使用 LRU 或其近似实现（伪-LRU）来平衡性能和硬件开销。

🎯 替换时的操作：

- 将新块的数据从主存中加载到被选中的缓存行；
- 替换行的标签字段更新为当前地址的标记位；
- 设置该行的有效位为 1；
- 如果使用的是写回策略，且被替换行为脏（dirty bit = 1），则需要**回写主存**。



### 6.4.4 全相连高速缓存

全相联高速缓存（fullyassociativecache）是由一个包含所有高速缓存行的组（即E=C/B)组成的。图6-35给出了基本结构

![6-5](assert/image-20250624142709604.png)

#### 1，组选择

![6-36](assert/image-20250624142742991.png)

#### 2.行匹配和字选择

![6-37](assert/image-20250624142827033.png)

因为高速缓存电路必须并行地搜索许多相匹配的标记，构造一个又大又快的相联高速缓存很困难，而且很昂贵。因此，***<u>全相联高速缓存只适合做小的高速缓存</u>***，例如虚拟内存系统中的翻译备用缓冲器（TLB），它缓存页表项（见9.6.2节）。



### 6.4.5 有关写的问题

假设我们要写一个已经缓存了的字w（写命中，write hit）。***<u>在高速缓存更新了它的的副本之后，怎么更新在层次结构中紧接着低一层中的副本呢？</u>***

- 最简单的方法，称为**直写（write-through）**，就是立即将的高速缓存块写回到紧接着的低一层中。虽然简单，但是直写的缺点是每次写都会引起总线流量。
- 另一种方法，称为**写回（write-back）**，尽可能地推迟更新，只有当替换算法要驱逐这个更新过的块时，才把它写到紧接着的低一层中。由于局部性，写回能显著地减少总线流量，但是它的缺点是增加了复杂性。**高速缓存必须为每个高速缓存行维护一个额外的修改位（dirtybit），表明这个高速缓存块是否被修改过。**

***另一个问题是如何处理写不命中***。

- 一种方法，称为写分配（write-allocate），加载相应的低一层中的块到高速缓存中，然后更新这个高速缓存块。写分配试图利用写的空间局部性，但是缺点是每次不命中都会导致一个块从低一层传送到高速缓存。
- 另一种方法，称为非写分配（not-write-allocate），避开高速缓存，直接把这个字写到低一层中。

直写高速缓存通常是非写分配的。写回高速缓存通常是写分配的。

### 6.4.6 一个真实的高速缓存等次结构的剖析

到目前为止：我们一直假设高速缓存只保存程序数据。不过，实际上，高速缓存既保存数据，也保存指令。

- 只保存指令的高速缓存称为icache。
- 只保存程序数据的高速缓存称为d-cache。
- 既保存指令又包括数据的高速缓存称为统一的高速缓存（unified cache）。

现代处理器包括独立的i-cache和d-cache。这样做有很多原因。有两个独立的高速缓存，处理器能够同时读一个指令字和一个数据字。i-cache通常是只读的，因此比较简单。通常会针对不同的访问模式来优化这两个高速缓存，它们可以有不同的块大小，相联度和容量。使用不同的高速缓存也确保了数据访问不会与指令访问形成冲突不命中，反过来也是一样，代价就是可能会引起容量不命中增加。

图6-38给出了IntelCorei7处理器的高速缓存层次结构。每个CPU芯片有四个核，每个核有自已私有的Lli-cache、L1d-cache和L2统一的高速缓存。所有的核共享片上 L3统一的高速缓存。这个层次结构的一个有趣的特性是所有的SRAM高速缓存存储器都在cpu芯片上。

![6-38](assert/image-20250624154009589.png)

![6-39](assert/image-20250624154032213.png)
