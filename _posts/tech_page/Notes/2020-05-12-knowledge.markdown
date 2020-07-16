---
layout: tech_post
title:  "计算机知识"
date:   2020-05-12 11:51:36 -0300
catalogue: Notes
tags: Notes 中文
description: 
---
## 目录
* TOC
{:toc}


### 寄存器

#### armv7构架

- R0-R7: 8个通用寄存器，类似8086中的AL，AH
- R8-R14: 7个备份寄存器，在不同模式下使用不同的物理寄存器
- R13: SP(stack pointer) 寄存器存放了一个指向栈顶的指针
- R14: lR(link register) 它保存了目前函数返回时下一个函数的地址，常与pc连用实现跳转。
- r15: PC(program ) 它存放了当前执行指令的地址, 与sp指针区别是pc存放的是程序指令的地址，sp存放数据地址。
- CPSR：(Current Program Status Register) 记录程序状态设定运行模式
	- Negative: 如果结果为负，该位为1，否则为0
	- zero: if(a==b)为真zero位为1，假为0 
	- carry: 加法进位1，否则0。减法借位0
	- overflow:
	
	
	- mode模式位: M[4:0] user:0b10000, fiq:0b10001, irq:0b10010, supervisor:0b10011....
<div align=center><img width = '60%' height ='60%' src ="/blog_photos/MPU/cpsr.jpg"/><p>Memory management</p></div>
- SPSR: (saved program status register)SPSR用于保存CPSR的状态，以便异常返回后恢复异常发生时的工作状态
### ROM 和 RAM

存储器在计算机中分为：主存，辅存和Cache

ROM 和 RAM 都是对对半导体言的且传统上讲都是计算机的内部存储器, RAM一般可以认为是手机，电脑的内存即掉电易失, ROM 是烧写在芯片内部的启动时运行。硬盘和u盘可以认为是辅存Secondary Storage。


### 计算机内存模型

#### 内存映射

虚拟内存是一种由MMU(memory management unit)与操作系统os共同实现的技术。简单讲虚拟内存可以把分散的物理地址块映射为连续的虚拟地址方便给进程使用。其中MMU主要负责将虚拟的地址转换为实际的物理地址,os负责管理虚拟地址空间。

下图展示了MMU映射的原理，其中MMU使用TLB(Translation Lookaside Buffer)作为cache缓冲存储刚刚使用过的页表。
- 当虚拟地址需要被转换为物理地址时，MMU首先会查找TLB缓存。如果命中则直接返回并进行(2.a)从主内存中读取数据 
- 如过TLB未能命中，MMU会在页表page table中进行全局查找(2.b)。如果页表中命中则更新TLB，此时返回第一步
- 不过有时在页表中也无法找到，这可能是由于访问了不合法的地址(此时程序会抛出segmentation fault)(3.b); 或者所要查找的页不在主内存中(如在硬盘中)。两种情况都会将控制从MMU传递给页管理器(page supervisor，内核程序)，对于第二种会产生页错误(3.c)，页管理器会进入硬盘将虚拟地址所需的内容恢复到主内存中(4)，并更新页表(5)
- 当所有物理主存耗尽时，页管理器不得不从内存中释放一部分空间并存入硬盘。 
Tips: 对于内存管理有多种算法，如Least Recently Used(LRU)

<div align=center><img width = '60%' height ='60%' src ="/blog_photos/MPU/mmu.jpg"/><p>Memory management</p></div>
参考[virtual paging and swapping](https://gabrieletolomei.wordpress.com/miscellanea/operating-systems/virtual-memory-paging-and-swapping/)

#### 内存模型


对于多任务的linux (32bit x86 architecture), 总是运行在4GB的虚拟地址空间上 2^32-1。起始的3 GB of 虚拟地址(i.e., from 0x00000000 to 0xBFFFFFFF) 被用于用户自己的进程 ，而最后的1 GB虚拟进程 (i.e., from 0xC0000000 to 0xFFFFFFFF) 被保留给 OS 内核如下图所示。

<div align=center><img width = '40%' height ='40%' src ="/blog_photos/MPU/memory_layout.jpg"/><p>Memory space</p></div>

在主存的应用进程中，程序和数据被分别存储在不同的地址段中
- <b>text</b>: 程序段，用于存储可执行的程序指令。一般当text段加载到内存中后便不可修改。
- <b>data()</b>： 数据段，用于存储已初始化的全局global和静态static变量。data段会进一步分为Read-only(Rodata)和read-write区。  
- <b>BSS(静态区)</b>：(Block Started by Symbol) 是未初始化的数据段，一般来说bss段在data段初始化后初始化，其中包含所有初始化为零或未初始化的变量(如 static int i)。bss不占用可执行文件空间，其内容由操作系统初始化，在初始化时bss段部分将会清零。  
- <b>stack栈</b>：向下增长，由程序自动分配释放(函数调用，函数返回)，存放函数的参数值、返回值和局部变量。遵循先进后出的原则。  
- <b>heap堆</b>：向上增长，动态分配且由程序员自行分配。堆空间紧接着bss段分配。由malloc/new,free/delete管理。

拓展：sp(stack point)栈指针指向栈顶的偏移地址。 当需用从函数A跳转函数B时，在汇编中常用push和pop作现场保护和现场恢复。PUSH {R0~R3, R12} 将R0，R1，R2，R3，R12寄存器中的值存入内存，当函数B运行完毕后使用pop{R0~R3, R12}将内存中的值取回寄存器，此时sp指针的地址回到A跳转B时的地址。



<div align=center><img width = '50%' height ='50%' src ="/blog_photos/MPU/program_in_memory2.png"/><p>Memory segmentation</p></div>

### IMX系列Uboot/裸机Boot(启动)流程

#### 上电流程
-1. cpu上电后根据引角配置选择boot_mode, 根据相应的boot_mode选择启动方式BOOT_MODE[1:0]：（01串口下载，10为内部boot）.当BOOT_MODE为10时,芯片会执行内部的boot ROM(96KB)代码,进行硬件初始化(一部分外设),然后从 boot设备中(根据BOOT_CFG系列引角，选择从SD，EMMC、NAND启动，启动后24个BOOT_CFG作为LCD_DATA复用)将代码拷贝出来复制到指定的 RAM 中,一般是 DDR.

-2. boot ROM 首先会读取IVT （image ector table）内部 Boot ROM 要求 IVT 应该放到指定的位置,不同的启动设备位置不同。下图Table.8.25展示了在不同设备中IVT存储的物理偏移地址。Table.8.26是ivt中的内容。

<div align=center><img width = '80%' height ='80%' src ="/blog_photos/MPU/IVT_table.png"/><p>Table. IVT offset nxp i.mx6ull</p></div>

-3. IVT 表中的内容中有几个比较重要： (1).entry: 说明了可执行程序的起始地址。这个连接地址可以是imx6内部的RAM仅128KB(0X900000~0X91FFFF)，也可是外部的RAM（DDR 起始于 0X80000000，终止256MB:0X8FFFFFFF,512:0X9FFFFFFF）. (2).dcd的地址(DeviceConfig Data) dcd中存储了部分寄存器的初始化值如外设的时钟、初始化 DDR。
<div align=center><img width = '80%' height ='80%' src ="/blog_photos/MPU/IVT_content.png"/><p>Table. IVT content of nxp i.mx6ull</p></div>


#### 程序初始化
裸机程序的初始化在arm构架中都是相似的。










 






