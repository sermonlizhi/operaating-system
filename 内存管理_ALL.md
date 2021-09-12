## 一、进程内存空间

#### 1.1 逻辑地址和物理地址

通过一个反编译的C语言文件查看逻辑地址

- 完整的C语言代码

  ```c
  #sample.c
  int sum(int x,int y)
  {
  	return x+y;
  }
  
  int main()
  {
  	sum(2,3);
  	return 0;
  }
  ```

- 使用gcc进行编译和汇编但不链接成可执行文件，得到目标二进制代码文件

  ```shell
  #生成二进制文件 sample.o
  gcc -g -c sample.c
  ```

- 使用objdump对sample.o进行反编译查看其汇编代码

  ```shell
  #反汇编指令  objdump -d sample.o
  sample.o：     文件格式 elf64-x86-64
  
  Disassembly of section .text:
  
  0000000000000000 <sum>:
     0:	55                   	push   %rbp
     1:	48 89 e5             	mov    %rsp,%rbp
     4:	89 7d fc             	mov    %edi,-0x4(%rbp)
     7:	89 75 f8             	mov    %esi,-0x8(%rbp)
     a:	8b 55 fc             	mov    -0x4(%rbp),%edx
     d:	8b 45 f8             	mov    -0x8(%rbp),%eax
    10:	01 d0                	add    %edx,%eax
    12:	5d                   	pop    %rbp
    13:	c3                   	retq   
  
  0000000000000014 <main>:
    14:	55                   	push   %rbp
    15:	48 89 e5             	mov    %rsp,%rbp
    18:	be 03 00 00 00       	mov    $0x3,%esi
    1d:	bf 02 00 00 00       	mov    $0x2,%edi
    22:	e8 00 00 00 00       	callq  27 <main+0x13>
    27:	b8 00 00 00 00       	mov    $0x0,%eax
    2c:	5d                   	pop    %rbp
    2d:	c3                   	retq   
  ```

  如上面的汇编代码所示，每条指令“:”前面的数字就是程序指令对应的逻辑地址，也称为虚拟地址。逻辑地址使用16进制表示，所有程序的逻辑地址都是从0开始的，后面跟着的数字是机器指令。

  以sum函数对应的汇编代码进行分析：

  ```shell
     0:	55                   	push   %rbp
     1:	48 89 e5             	mov    %rsp,%rbp
     4:	89 7d fc             	mov    %edi,-0x4(%rbp)
     7:	89 75 f8             	mov    %esi,-0x8(%rbp)
     a:	8b 55 fc             	mov    -0x4(%rbp),%edx
     d:	8b 45 f8             	mov    -0x8(%rbp),%eax
    10:	01 d0                	add    %edx,%eax
    12:	5d                   	pop    %rbp
    13:	c3                   	retq  
  ```

  0x55指令对应一个字节，所以第二条指令的逻辑地址等于第一条指令逻辑地址+1=1，第二条指令0x48 89 e5对应3个字节，所以第三条指令的逻辑地址等于4，依次类推。

物理地址是内存单元看到的地址，是指令和数据真实存在的内存地址，而逻辑地址是面向程序而言的。CPU在执行指令的时候，会先将逻辑地址经过一个MMU(Memory Management Unti)的硬件设备转换成物理地址，然后从物理地址取出指令执行

#### 1.2 进程的内存映像

以32位的机器为例，地址总线为32位，可寻址的最大内存空间是2^32Bytes，即4GB。每个运行的程序可以获得一个4GB的逻辑地址空间，这个地址空间被分为两部分：内核空间和用户空间，其中用户空间分配0x00000000到0xC0000000共3G的地址，而内核空间分配了0xC0000000到oxFFFFFFFF高位的1GB地址。

而用户进程的逻辑地址内容就是进程内存映像，一共包含四个部分的内容：

- text代码段：只读，存放代码的指令，上面反汇编后的文件中section .text表示的就是代码段
- data数据段：存放全局或静态变量
- heap堆：运行时(Run time)分配的内存(如用malloc函数申请内存)
- stack栈：存放局部变量和函数返回地址

下面通过代码查看各个部分在内存中的情况：

- 数据段

  ```c
  #dataSegment.c
  int global_var = 5;
  
  int main(int argc, char const *argv[])
  {
  	static int static_var = 8;
  	return 0;
  }
  ```

  ```shell
  lizhi@Dog-li:~/test/c/memory$ gcc -g -c dataSegment.c
  lizhi@Dog-li:~/test/c/memory$ objdump -s -d dataSegment.o
  #部分代码如下
  dataSegment.o：     文件格式 elf64-x86-64
  
  Contents of section .text:
   0000 554889e5 897dfc48 8975f0b8 00000000  UH...}.H.u......
   0010 5dc3                                 ].              
  Contents of section .data:
   0000 05000000 08000000  
  ...... 
  Disassembly of section .text:
  
  0000000000000000 <main>:
     0:	55                   	push   %rbp
     1:	48 89 e5             	mov    %rsp,%rbp
     4:	89 7d fc             	mov    %edi,-0x4(%rbp)
     7:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
     b:	b8 00 00 00 00       	mov    $0x0,%eax
    10:	5d                   	pop    %rbp
    11:	c3                   	retq 
  ```

  如上所示，通过objdump -s将详细的内存信息都打印出来，Contents of section .text下展示的是代码段的指令对应的机器码，与Disassembly of section .text是一致的；而Contents of section .data展示的是数据段存放的值，代码中global_var和static_var分别对应5和8存放在数据段里面

- 堆栈段

  堆栈数据只有在程序运行时才能在内存中看到，所以下面代码让程序处于休眠状态

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  
  int global_var = 5;
  
  int main(int argc, char const *argv[])
  {
  	static int static_var = 8;
  	int local_var = 10;
  	int* p = (int*)malloc(100);
  
  	printf("the global_var address is %1x\n", &global_var);
  	printf("the static_var address is %1x\n", &static_var);
  	printf("the local_var address is %1x\n", &local_var);
  	printf("the address with the p points to %1x\n", p);
  
  	free(p);
  
  	sleep(1000);
  
  	return 0;
  }
  ```

  ```shell
  #执行程序
  lizhi@Dog-li:~/test/c/memory$ ./heapStack 
  the global_var address is 55bdc4b4a010
  the static_var address is 55bdc4b4a014
  the local_var address is 7ffd5a55169c
  the address with the p points to 55bdc60be260
  
  
  #查找进程对应的
  lizhi@Dog-li:~$ ps -e |grep 'heapStack'
    7322 pts/0    00:00:00 heapStack
  
  #通过下面命令可以看到进程的相关段的大小限制以及进程的其他信息
  lizhi@Dog-li:~$ cat /proc/6781/status
  ......
  VmData:	     176 kB
  VmStk:	     132 kB
  VmExe:	       4 kB
  VmLib:	    2108 kB
  VmPTE:	      52 kB
  VmSwap:	       0 kB
  ......
  
  #通过下面命令可以看到当前正在运行进程的映射信息(程序各部分的地址段)
  lizhi@Dog-li:~$ cat /proc/6781/maps
  |   地址范围(from_to)    | 权限| offset |设备ID| iNode|
  55bdc4949000-55bdc494a000 r-xp 00000000 08:01 548264     /home/lizhi/test/c/memory/heapStack
  55bdc4b49000-55bdc4b4a000 r--p 00000000 08:01 548264     /home/lizhi/test/c/memory/heapStack
  55bdc4b4a000-55bdc4b4b000 rw-p 00001000 08:01 548264     /home/lizhi/test/c/memory/heapStack
  55bdc60be000-55bdc60df000 rw-p 00000000 00:00 0          [heap]
  7fee880d7000-7fee882be000 r-xp 00000000 08:01 923849     /lib/x86_64-linux-gnu/libc-2.27.so
  7fee882be000-7fee884be000 ---p 001e7000 08:01 923849     /lib/x86_64-linux-gnu/libc-2.27.so
  7fee884be000-7fee884c2000 r--p 001e7000 08:01 923849     /lib/x86_64-linux-gnu/libc-2.27.so
  7fee884c2000-7fee884c4000 rw-p 001eb000 08:01 923849     /lib/x86_64-linux-gnu/libc-2.27.so
  7fee884c4000-7fee884c8000 rw-p 00000000 00:00 0 
  7fee884c8000-7fee884ef000 r-xp 00000000 08:01 923821     /lib/x86_64-linux-gnu/ld-2.27.so
  7fee886d9000-7fee886db000 rw-p 00000000 00:00 0 
  7fee886ef000-7fee886f0000 r--p 00027000 08:01 923821     /lib/x86_64-linux-gnu/ld-2.27.so
  7fee886f0000-7fee886f1000 rw-p 00028000 08:01 923821     /lib/x86_64-linux-gnu/ld-2.27.so
  7fee886f1000-7fee886f2000 rw-p 00000000 00:00 0 
  7ffd5a532000-7ffd5a553000 rw-p 00000000 00:00 0          [stack]
  7ffd5a5d5000-7ffd5a5d8000 r--p 00000000 00:00 0          [vvar]
  7ffd5a5d8000-7ffd5a5d9000 r-xp 00000000 00:00 0          [vdso]
  ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0  [vsyscall]
  ```

  上面的堆栈信息，显示了程序各个部分的Address(逻辑地址空间)，Permissions，Offset，Device，iNode，PathName

  - Address：程序映射占用的起始地址和结束地址
  - Permission：readable，writable，executable，private，shared
  - Offset：基于逻辑内存开始地址的在文件中的偏移量，并不是每个逻辑地址都是从文件映射，所以有的偏移量为0
  - iNode：相关文件的编号

  `Offset和iNode在文件系统中会详细介绍`

  上面的堆栈信息中，第一行的权限是可读可执行，所以为代码段(text segmen)；第三行的权限为可读可写，故为数据段(data segment)，结合我们执行程序打印的各个变量和指针的内存地址，可以看到全局变量和静态变量的地址对应在数据段的地址范围，而局部变量的地址对应于栈的地址范围，指针的地址对应于堆的地址范围。

## 二、内存管理

#### 2.1 内存介绍(Main Memory)

`内存一般称之为主存,而磁盘等外部存储器称之为辅存,它们的硬件构造是完全不同的`

- Main Memory is certral to operation of a modern computer system

- Memory contains of a large array of bytes,each with its own address.

  内存包含了很多字节序列，每个字节都有一个地址与之对应

- The CPU fetches(获取) instructions from memory according to the value of the program counter(PC).These instructions may cause addtional loading from and storing to specifi memory address.

  `PC寄存器：存放下一条要执行的指令的内存地址`

  CPU根据PC寄存器中一条指令的地址从内存中获取指令，这些指令可能造成额外地从指定内存获取和向内存写入的动作

- A typical instruction-execution cycle,for example,first fetches an instruction from memory.The instruction is then decoded and may cause operands to be fetched from memory.After the instruction has been executed on the operands,result may be stored back in memory.

  经典的指令执行周期：首先从内存中获取一条指令，然后对指令进行解码，这个过程可能要再从内存中获取操作数，最后使用操作执行指令，指令运行的结果可能再被写入到内存

#### 2.2 高速缓存(CACHE)

- **高速缓存**是一种存取比内存快，但容量比内存小的多的存储器，它可以加快访问物理内存的相对速度

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828100001963.png" alt="image-20210828100001963" style="zoom:50%;" />

  现代操作系统中，一般分为三级缓存，一级和二级缓存一般集成在CPU内部，缓存的级数越高，其容量也就越大，与之对应的存取速度也就越慢

  对于多级别的缓存，数据一致性将是一个很重要的问题

#### 2.3 内存数据保护

- 用户进程不可以访问操作系统内存数据，以及用户进程空间不能相互响应

  - 通过硬件来实现，操作系统一般不干预CPU对内存的访问

    base register：基址寄存器

    limit register：限长寄存器

  - 上述两个寄存器的值只能被操作系统的特权指令加载(Kernel模式下执行)

- 地址保护

  策略：与限长寄存器的limit值进行比较

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828124110003.png" alt="image-20210828124110003" style="zoom:50%;" />

#### 2.4 地址空间与转换

`在前面进程内存空间部分，已经介绍过了逻辑地址和物理地址的概念`

所有逻辑地址的集合称为逻辑地址空间，这些逻辑地址对应的所有物理地址集合称为物理地址空间

在一个程序中，每条指令的逻辑地址就是与第一条指令之间的相对偏移，所以逻辑地址也成为偏移地址(可以直接理解为偏移量)

每条指令的物理地址=程序起始指令的物理地址(基址)+逻辑地址(偏移量)

基址存放在基址寄存器(base register)

**地址转换：**由逻辑地址转换成物理地址

`现代操作系统都是在程序运行时进行地址转换`

##### 2.4.1 内存管理单元MMU

- Memory-Management Unit完成逻辑地址到物理地址运行时的转换工作

  MMU基于重定位寄存器(relocation register)或基址寄存器(base register)工作

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828104431259.png" alt="image-20210828104431259" style="zoom:50%;"/>

  physical address = 1200(base address) + 120(logical address) = 1320

#### 2.5 连续内存分配

In contiguous memory allocation,each process is contained in a single section of memory that is contiguous to the section containing the next process.

在连续内存分配中，每个进程包含一个单独的内存块且与下一个进程的内存块是连续的。每个进程所占用的内存也是连续的，

##### 固定大小分区(FIXED-SISED PARTTION)

Memory is divided to serval fixed-sised partitions.Each partition may contain exactly one process.

操作系统将内存划分为数个大小不等的分区，每个分区恰好存放一个进程，同时操作系统需要维护一张内存分区表，记录每个分区的编号、基址、分区大小以及是否可用等信息

缺点：灵活性太差，操作系统对于最大分区分配多大内存没有标准，可能导致一个很大的进程因为需要更大的内存而无法使用；造成内部碎片，一个分区大小为10字节分配给了一个大小为6字节的进程，其中剩下的4个字节将无法被使用

内存回收时，是需要将内存分区表中信息改为可用即可

##### 可变分区(VARIABLE-PARTITION)

- In the variable-parition scheme,the operating system keeps two tables indicating which parts of memory are avariable and which are occupid

  在可变分区中，操作系统需要维护两张表，分别指明内存是否可用；两张表都包含了基址、内存大小等信息

- Initially,all memory is avariable for user processes and is considered one large block of avariable memory,a hold

  初始的时候，所有的内存对于用于进程都是可用的，被看成是一大块可用内存，即一个孔

- Eventually,as you willl see,memory contains a set of holes of various sizes.

  最后，内存将会被分成一系列各种大小的孔

`一开始内存还没有使用,可以看成一个孔,然后操作系统为每个程序分配满足运行大小的内存块,当一个程序运行完退出之后,之前分配的内存块就变成可用的了,操作系统就会在可用表里面记录该孔的信息(基址、内存大小等),当再有程序需要分配内存时,如果该孔的大小满足,就可以为新程序分配该孔`

**动态存储分配策略：**

- 首次适应

  操作系统查找可用内存表时,为程序分配首个满足程序的足够大的孔，效率最高

- 最佳适应

  分配最小的可以满足程序的孔，浪费最小

- 最坏适应

  分配最大的孔，产生的剩余孔更可能被利用；这种方案的最大问题是每次都使用最大的孔，如果有一个进程需要足够大的孔时，将无法为其分配孔导致无法执行

可变分区的内存回收时，从已用分区表中删除分区信息，同时插入到可用分区表

`这两种连续分配方案的地址转换方式是相似: 物理地址=基址+逻辑地址`

##### 碎片

Fragmentation：some little piece of memory hardly to be used(很小的内存片,难以利用)

- internal fragmentation(内部碎片)

  相对于固定分区来说，例如一个分区有10K字节大小，分配给了一个只需要5K字节的程序，剩下的5K字节仍然属于该分区，处于分区内部，但不能分配给其他程序使用

- external fragmentation(外部碎片)

  相对可变分区来说，例如一个分区有10K字节大小，分配给一个只需要5K字节的程序后，剩下的5K字节形成了一个新的可用分区记录在表中，当有程序需要时，可以继续分配

  `对于可变分区产生的碎片,考虑进行压缩然后使用,将很多小的碎片分区合并成一个大的分区,但碎片压缩一定要考虑到对性能的影响`

## 三、分段与分页

#### 3.1 动机(MOTIVATION)

- Solution to fragmentation：permit the logical address space of processes to be noncontigunous

  分段和分页作为碎片问题的一个解决方案，允许进程的逻辑空间地址不是连续的

- The view of momory is different between

  - logical(programmer's)：a variable-sized segments
  - physical：a linear array of bytes

  在开发人员眼中，内存就是一系列大小不等的段(包含变量、函数等)，对应于逻辑地址，而从物理地址的角度看，内存就是一个线性的字节数组

- The hardware could provide a memory mechanism that mapped the logical view to the actual physical memory

  需要硬件提供一种机制，将逻辑试图映射成实际的物理内存

#### 3.2 分段(可变分区)

从开发人员角度看，程序由主函数和一组其他函数构成，包含各种数据结构(变量、结构体、对象、数组等)，所有的模块都是通过名字来引用的。

每个模块对应内存中大小不等的段，每个段有专门的用途，段的大小和用途相关

`段和段之间不必连续存放(离散)`

假设一个程序有A、B、C三个函数，将这三个函数分成3段，每个段内指令的逻辑地址都是从0开始，但是每一段的基址却是不同的

分段硬件：段基址(base register)、段限长(limit register)、段表

其中段表包含了段号、基址和限长

![image-20210828160416804](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828160416804.png)

逻辑地址由段号和段内位移构成

![image-20210828160244676](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828160244676.png)

 地址转换的时候，根据逻辑地址的段号找到段表里面的记录，然后判断段内位移是否查过限长，如果不超过则计算物理地址

#### 3.3 分页(固定分区)

-  基本方法

  分页是基于连续内存分配中固定分区实现的，只不过分页是将内存分成较小的大小一致的分区，每个分区称为帧或页框

  然后操作系统将应用程序分成多个与页框大小相等的页，然后给应用程序的每页分配一个页框的内存，同时用一张页表记录来记录程序的页号与内存页框号的映射关系

  ![image-20210828163308025](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828163308025.png)

- 逻辑地址

  分页中的逻辑地址使用页号和页内位移来表示

  ![image-20210828163528287](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828163528287.png)

- 物理地址

  首先根据逻辑地址的页号找到页表中对应的页框号，因为内存中每个页框的大小是固定的，可以直接判断程序指令逻辑地址对应的页内偏移是否超过限长，然后再进行地址转换

  物理地址 = 页框号*页框大小 + 页内位移

  physical address = frame_no * pagesize + offset

- 页面大小

  - 页大小(或页框大小)是由硬件定义的，每页的大小是2的幂次方，在512字节和1GB之间变化，依赖于电脑结构。常用大小为4K字节

  - 页的大小选择为2的幂次方是为了方便地址转换，不用进行算术运算

  - 如果一个程序的逻辑地址空间大小为2^m，页的大小为2^n字节，那么一个逻辑地址高位的m-n比特表示页号，低位的n比特表示页内位移

    ![image-20210828170652297](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828170652297.png)

    

#### 3.4 分页与分段的区别

|                分段                |                  分页                  |
| :--------------------------------: | :------------------------------------: |
|           信息的逻辑单位           |             信息的物理单位             |
|            段长是任意的            |             页长由系统确定             |
| 段的起始地址可以从主存任意地址开始 | 页框起始地址只能以页框大小的整数倍开始 |
| (段号、段内位移)构成了二维地址空间 |   (页号、页内位移)构成了一维地址空间   |
|           会产生外部碎片           |    消除了外部碎片，但会出现内部碎片    |

## 四、页表

#### 4.1 PAGE TABLE

- The operating system maintains a copy of the page table fro each process

- This copy is used to translate logical address to physical address

- It is also used by the CPU dispatcher to define the hardware page table when a process is to be allocate the CPU

  当进程被分配CPU时，CPU的调度程度使用它来定义硬件页表

- Paging therefore increase the context-switch time

  分页将造成进程上下文切换的一个开销，进程被分配CPU后，每次CPU都需要调入页表

#### 4.2 分页

##### 4.2.1 逻辑地址

在分页里面介绍了分页的逻辑地址是由页号和页内位移组成，但它是一个一维的结构

假设：page_size = 4 byte，程序有8条指令，每个指定对应一个字节，操作系统会将程序分成两个页，页号分别0和1，页内位移从0到3

![image-20210828174639458](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828174639458.png)

从上面的图可以看出，分页后的逻辑地址对应的值与分页前程序每条指令对应的逻辑地址是一样的，整个程序的逻辑地址可以从0开始表示

##### 4.2.2 Frame Table

在分页中，操作系统会维护一张页框状态表Frame Table，记录哪些页框已经被用，哪些可用

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828175316465.png" alt="image-20210828175316465" style="zoom:50%;" />

##### 4.2.3 Page Table

页表Page Table是使用一个一维数组进行存储，其中数组的下标表示页号，索引值表示页框号

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828175904688.png" alt="image-20210828175904688" style="zoom:50%;" />

##### 4.2.4 Page Size

如前面3.3分页中讲到的页面大小，如果逻辑地址长度为mbits，页面大小：2^nBytes

- 页内位移：n bits
- 页号：m-n bits

获取Linux系统页大小的命令：

```shell
lizhi@Dog-li:~$ uname -m
x86_64
lizhi@Dog-li:~$ getconf PAGESIZE
4096
#对应的页面大小的相关数值
m = 64  n = 12   
理论上页号占52bits，实际操作系统页号占 48-12=36bits
```

#### 4.3 快表

#####  4.3.1 PTBR

- The page table is kept in main memory,and a page-table base register(PTBR) points to the page table

  把页表存在主存里面，然后用一个页表基址寄存器存一个指向进程页表的指针

- Changing page tables requires changing only this one register,substantially reducing context-switch time

  每次进程上下文切换时，不需要来回加载保存和加载页表信息，只需要更改页表基址寄存器的指针即可

- With this scheme,two memory accesses are needed to access a byte(one for the page-table entry,one for the byte)

  用这种方案，每获取一个指令的需要访问两次内存，一次是获取页表，另一次是根据页表转换后的物理地址获取指令

##### 4.3.2 TLB(快表)

- TLB(Translation Look-aside Buffer) is a kind of small,fast-lookup hardware cache.It is used with page tables in the following ways

  TLB是一种很小、查询速度很快的专用硬件缓存，配合页表使用

  - The TLB contains only a few of the page-table entries

    TLB只包含一个进程一部分页表内容

  - When a  logical address is generated by the CPU,its page number is presented to the TLB

    当CPU为程序生成一个逻辑地址时，CPU将首先检查TLB种是否有对应的页号

  - If the page number is found,its frame number is immediately available and is used to access memory

    如果TLB中有这个页号，则对应的页框号当即可用去访问内存中的指令

  - If TLB miss,a memory reference to the page table must be made

    如果TLB中没有这个页号，则需要访问内存获取页框号，然后再访问内存中对应的指令(需要访问两次内存)

- PAGING TLB

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828193735770.png" alt="image-20210828193735770" style="zoom:40%;" />

- TLB HIT RATIO(TLB命中率)

  The percentage of times that the page number of interest is found in the TLB is called the hit ratio

  从TLB中找到页号次数的备份比称为命中率

  An 80-percent hit ratio,for example,means that we find the desired page number in the TLB 80 percent time,If it takes 100 nanoseconds to access memory,find the effective memory-access time

  effective access time = 0.8 * 100 + 0.2 * 200 = 120ns

  访问TLB就能获取页号的概率为80%，则平均内存访问时间就是120ns

#### 4.4 基于页的保护与共享

##### 4.4.1 保护

为了防止地址转换(CPU计算程序页号)时出现异常，可在页表的每个条目设置一个“valid-invalid”比特位，用于表示该页的有效性

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828195346514.png" alt="image-20210828195346514" style="zoom:50%;" />

这个方法可以扩展以提供更好的保护级别，如：”只读“、”读写“、”可执行“

##### 4.4.2 共享(pure code)

对于某些只读的代码，多个应用程序都会用到，则没必要在每个程序进程中为这些代码维护一份页表信息，只需要在内存中为这些代码维护一份页表信息即可，但这些页表项是只读的

#### 4.5 多级页表

##### 4.5.1 页表大小

假设CPU是32bits，采用的逻辑地址是32bits，那么进程的逻辑地址空间大小为2^32Bytes，即4GB

- 若页面大小是4K Bytes，则一个进程最多被分成1M个页面，也就是说进程的页表最多有1M个页表项
- 若每个页表项占用4Bytes，则每个页表最多占用4MBytes连续内存空间(1K个连续页框)

因为我们需要把页表也放在内存的页框中，但又没有这么大的连续内存空间，只能把页表进行拆分后放入页框中，那么我们就需要记录哪些页表存放在了那个页框里面，如上面的假设，每个页框大小为4KBytes，而每个页表项又占用4Bytes，那么一个页框可以存放1K个页表项，于是，就将页表项以1000个为单位分成页表页，即每个页表页存放1000个页表项，页表页的序号从0开始

**页表页的结构如下：**

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828202612695.png" alt="image-20210828202612695" style="zoom:75%;" />

**逻辑地址**

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828202931179.png" alt="image-20210828202931179" style="zoom:75%;" />

计算物理地址时，根据页表页号从页表页中找到对应的页框号，根据页框号，将页表从内存中取出，在根据页号从页表中找到对应的页框号，根据页框号和页内位移计算出物理地址

`因为页表页也需要存放在页框中,如果页表页所需要的连续空间大小超过了页框大小,需要再对页表页进行拆分,形成三级页表`

x86-64架构CPU采用的四级页表方案：

![image-20210828203611188](https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828203611188.png)

正如我们在4.2.4Page Size中说到的，操作系统采用36bits来表示页号，36bits已经完全够用了

## 五、虚拟内存

#### 5.1 局部性原理

在上面第四部分的页表中，通过将部分页表放入到TLB中加快访问速度，这些页表称为快表，但哪些页表应该放入到TLB(Cache)中呢？

##### 5.1.1 缓存Cache(一般的)

`CPU访问数据的时候，首先到相应的缓存里面查找数据，如果缓存中没有，则访问内存获取，同时将从内存中获取到的数据存入到缓存中`

**Cache结构**

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828210413387.png" alt="image-20210828210413387" style="zoom: 75%;" />

如上图所示，假设Cache的每一格是8Bytes，每一个缓存行是64Bytes，Cache的读写是以缓存行(CacheLine)为单位的，当CPU从内存读取数据时，并不只是将所需要的数据读取到Cache中，而是将读取到的数据的起始位置到后面64Bytes的数据当作一个缓存行写入到内存中。例如内存中放有一个64Bytes*8的二维数组，CPU每次行列顺序读取数组，读一次内存就可以将一行的数据加载进内存，这样可以加快CPU的访问速度

**修改缓存数据**

读缓存不会有数据不一致的情况，但是写缓存，就会导致缓存里面的数据与内存中的数据不一致，又如下两种方案：

- Write through(直接写)

  修改缓存数据的同时修改内存数据，这种方式会导致CPU执行效率变低

- Write back(回写)

  先只修改缓存数据，直到该数据要被清除缓存时在修改内存中的数据，保证数据最终一致性

**清除缓存数据**

缓存的容量很小，当缓存满的时候，就需要将缓存中的数据淘汰，装入新的数据

对于淘汰算法，将会在下面虚拟内存的页面置换算法中详解介绍

##### 5.1.2 局部性原理

对于一个程序而言，有些代码可能永远都不会被访问，而有些局部数据会经常访问，但是将哪些数据放入内存中？

局部性原理提供了以下方案

- 时间局部性(Temporl locality)

  如果某个信息这次被访问，那它有可能在不久的未来被多次访问

- 空间局部性(Spatial locality)

  如果某个位置的信息被访问，那和它相邻的信息也有可能被访问到

- 内存局部性(Memory locality)

  访问内存时，大概率会访问连续的块，而不是单一的内存地址，其实就是空间局部性在内存上的体现

- 分支局部性(Branch locality)

  计算机大部分指令是顺序执行，顺序执行和非顺序执行的比例大致是5:1，顺序执行的指令是空间连续的

- 等距局部性(Equidistant locality)

  如果某个位置被访问，那和它相邻等距离的连续连续地址极有可能被访问到

#### 5.2 虚拟内存

##### 5.2.1 部分装入和部分对换

- 部分装入
  - 进程运行时仅加载部分进入内存，而不必全部装入
  - 其余部分暂时存放在swap space
- 部分兑换
  - 可以将进程部分对换出内存，用以腾出内存空间
  - 对换出的部分暂时存放在swap space

缓存—主存—辅存之间的关系

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210828220553570.png" alt="image-20210828220553570" style="zoom:50%;" />

##### 5.2.2 虚拟内存概念

- Virtual Memory is a technique that allows the execution of processes that are not completely in Memory

  虚拟内存是一种允许执行进程无需完全在内存中技术(**部分装入**)

- One major advantage of this scheme is that programs can be larger than physical memory

  这种方案最大的优势是程序总内存可以比实际的物理内存大(4G内存可以运行8G的程序)

- Further,virtual memory abstracts main memory into an extremely large,uniform array of storage,separating logical memory as viewed by the user from physical memory

  虚拟内存技术将主存抽象成一个很大的、统一的数组，将逻辑内存和物理内存分离开来(对用户包括开发人员来说，只需要关注逻辑内存，无需关注物理内存)

- This technique frees programs from the concerns of memory-storage limitations

  这种技术将程序从内存存储的限制中解放出来，程序不需要再关心内存不够用的情况

#### 5.3 请求调页

在上面4.5节多级页表中展示了Linxu系统采用的是四级页表的方式，而不是分段的方式进行内存管理

虚拟内存技术是基于分页来实现的

##### 5.3.1 DEMADN PAGING

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829094036793.png" alt="image-20210829094036793" style="zoom:50%;" />

- With demand-paged virtual memory,pages are loaded only when they are demanded during program execution

  在请求调页的虚拟内存中，只有当执行程序请求的时候，页面才会被加载进内存

- Pages that are never accssed are thus never loaded into physical memory

  从来没有被请求访问的页面也因此从来不会被加载进内存

如上图所示，物理内存中只存了程序A、C、F三个页，当程序需要H页时，发现页表中页表号7对应的页框为空，则需要将H页从磁盘中加载进物理内存的页框中，同时更新页表中对应的页框号和valid-invalid bit，这整个过程就称为调页。

##### 5.3.2 请求调页步骤

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829095551767.png" alt="image-20210829095551767" style="zoom:50%;" />

上图中的load M对应的是Logic Memory，也是一个page存储在内存中

1、根据逻辑地址表从页表中获取对应的页表项

2、页表项为i(invalid)，则陷入page fault(缺页中断)

3、操作系统收到中断后，查找磁盘中的请求页

4、将磁盘上的页加载进主存中空闲的页框

5、更新页表的页表项

6、返回逻辑地址对应的指令

##### 5.3.3 请求调页的性能

- 假设访问内存时间为ma，处理一次缺页中断的时间记作page fault time，令p为缺页中断的出现几率，则有效访问时间的计算公式：

  effective access time = ma * (1-p) + page fault time * p

- 如果ma = 200ns，page fault time = 8ms，p=0.001，则

  effective access time = 8200ns

`缺页中断率P对性能影响重大`

#### 5.4 页面置换算法

不论是内存还是缓存，都有存储空间被用完的时候，如果这时候又有新的数据需要加载进内存或缓存，那么需要将哪些数据从内存或缓存中移除呢，这就是页面置换算法需要解决的问题

##### 5.4.1 页面置换

当进程在执行过程中出现了缺页中断，在请求调页的时候发现内存已经没有空闲的页框可用，操作系统在此时会做出一个处理：页面置换

将内存中的页从页框移除转移到磁盘中存储，这个过程就是前面说到的swap out过程

**页面置换过程**

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829104553381.png" alt="image-20210829104553381" style="zoom:50%;" />

1、将内存页框中的页交换至磁盘

2、更新该页在页表中的状态(变成不可用)

3、将请求的新页从磁盘加载进内存

4、更新新页在页表中的状态为可用

##### 5.4.2 FIFO(先进先出)

- 总是淘汰最先进入内存的页面，因为它呆在内存中的时间最久

- 假设操作系统给一个进程分配了3个页框，按照下面的页号顺序访问，一共需要访问20个页，计算缺页中断率P

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829105330975.png" alt="image-20210829105330975" style="zoom:50%;" />

  进程一共只有3个页框，前面页号7、0、1都是通过缺页中断加载进内存的，当访问第四个2号页时，页框中没有，发生缺页中断，按照FIFO的规则，需要将7号页交换出内存，将2号页加载进内存；当访问第五个0号页时，内存中有该页则直接访问，；当访问第六个3号页时，再次发生缺页中断…………依次类推直到访问到最后一个页，一共发生了15次缺页中断

  缺页中断率P = 15 / 20 = 75%

  `这种方案是最简单、最容易被实现的`

##### 5.4.3 OPTIMAL(最优)

- 总是淘汰将来最长时间不会再使用的页面

  按照上面FIFO的例子计算缺页中断率P

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829105330975.png" alt="image-20210829105330975" style="zoom:50%;" />

  前面三个页7、0、1加载进内存发生三次缺页中断，当需要使用2号页时，则根据后面需要访问页的面顺序，发现0页号在2号页后面就会被使用，然后是1号页被使用，最后才是7号页被使用，那么按照OPTIMAL规则应该将7号置换出内存，将2号页加载进内存…………依次类推直到访问到最后一个页，一共发生了9次缺页中断

  缺页中断率P = 9 / 20 = 45%

  `这种方案现实中是无法实现的，操作系统没办法预测将来需要哪些页`

##### 5.4.4 LRU(最近最少使用)

- LRU(LEAST RECENT UNUSED)总是淘汰最近最少使用的页面

  根据以前的页面使用的情况，将最近最少使用的页面置换出内存

  按照上面FIFO的例子计算缺页中断率P

  <img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829105330975.png" alt="image-20210829105330975" style="zoom:50%;" />

  前面三个页7、0、1加载进内存同样发生三次缺页中断，当需要使用2号页时，按照LRU的规则，发现7号页时最近最少使用，将其置换出内存，将2号页加载进内存，接下来访问0号页，在内存中可以直接访问，此时内存的页框中有0、1、2三个页，当需要访问3号

  页时，发现1号页是最近最久没有使用的，将其置换出内存…………以此类推直到访问到最后一个页，一共发生了12次缺页中断

  缺页中断率P = 12 / 20 = 60%

  `另外还有一个算法Clock(second Chance)，该算法是基于LRU实现的`

#### 5.5 系统抖动

##### 5.5.1 THRASHING(抖动)

- If the process does not have the number of frames it need to support pages in active use,it will quickly page-fault.At the point,it must replace some pages.However,since all its pages are in active use,it must replace a page that will be needed again,and again,replacing pages that it must bring back in immediately

  如果进程没有足够的页框去存放常用页，就会经常陷入缺页中断，这时候，必须将其中一些页置换出内存，然而所有的页都是常用，下次又要使用刚刚置换出去的页，它又要通过缺页中断被再次加载进内存，往复循环。

- This high paging activity is called thrashing.A process is thrashing if it is spending more time paging than executing

  这种高频率的调页称为抖动，进程抖动会造成CPU处理消耗更多的时间来处理缺页中断而不是执行程序

##### 5.5.2 系统抖动的原因

- 并发进程数量过多

  下图展示了操作系统多道程序的数量与CPU使用率的关系，一开始随着并发进程数量的增加，CPU使用率越来越高，但得到一定的程度之后，进程数继续增加，每个进程分配的页框数就变小了，不足以存放常用页，就会导致频繁的缺页中断，CPU利用率急剧下降

- 进程页框分配不合理

  假设进程A常用页有5个，但操作系统只给其分配了3个页框，也会导致频繁的缺页中断，消耗CPU时间

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829120108218.png" alt="image-20210829120108218" style="zoom:50%;" />

##### 5.5.3 解决方案

PFF(page fault frequency) 称作页面故障率，基于下图的数据，可以实施一个防止抖动的策略：操作系统动态调节分配给进程的页框数量

<img src="https://raw.githubusercontent.com/sermonlizhi/picture/main/image-20210829121356744.png" alt="image-20210829121356744" style="zoom:50%;" />

上图展示了给进程分配的页框数量和缺页中断率的关系，给进程分配的页框越多，缺页中断率就越低，但也不能把所有页框都分配给少量的几个进程导致其他进程出现系统抖动，于是操作操作系统会把进程的页框数量控制在上图的upper bound 和lower bound之间，将整个系统中进程的缺页中断率控制在一个可控的范围。

##### 5.3.4 CONCLUDING REMARKS(结论)

- 系统抖动和缺页中断对性能是一个很大的影响
- 当前最佳实践就是给电脑足够大的内存(加内存条)，避免系统抖动和页面置换
- 将进程工作的所有页面集合全部装进内存中，可以给用户带来良好的用户体验

## 六、大容量存储

