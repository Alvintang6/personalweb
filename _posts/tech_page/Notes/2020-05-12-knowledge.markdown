---
layout: tech_post
title:  "计算机知识"
date:   2020-05-12 11:51:36 -0300
catalogue: Notes
tags: Notes
description: 
---
## 目录
* TOC
{:toc}

### ROM 和 RAM

存储器在计算机中分为：主存，辅存和Cache

ROM 和 RAM 都是对对半导体言的且传统上讲都是计算机的内部存储器, RAM一般可以认为是手机，电脑的内存即掉电易失, ROM 是烧写在芯片内部的启动时运行。硬盘和u盘可以认为是辅存。


### 计算机内存模型

#### 内存映射


<div align=center><img width = '60%' height ='60%' src ="/blog_photos/MPU/mmu.jpg"/><p>Memory management</p></div>
参考[virtual paging and swapping](https://gabrieletolomei.wordpress.com/miscellanea/operating-systems/virtual-memory-paging-and-swapping/)

### IMX系列Uboot/裸机启动流程

-1. cpu上电后根据引角配置选择boot_mode, 根据相应的boot_mode选择启动方式BOOT_MODE[1:0]：（01串口下载，10为内部boot）.当BOOT_MODE为10时,芯片会执行内部的boot ROM代码,进行硬件初始化(一部分外设),然后从 boot设备中(根据BOOT_CFG系列引角，选择从SD，EMMC、NAND启动，启动后24个BOOT_CFG作为LCD_DATA复用)将代码拷贝出来复制到指定的 RAM 中,一般是 DDR.

-2. boot ROM 首先会读取IVT （image ector table）内部 Boot ROM 要求 IVT 应该放到指定的位置,不同的启动设备位置不同。下图Table.8.25展示了在不同设备中IVT存储的物理偏移地址。Table.8.26是ivt中的内容。

<div align=center><img width = '80%' height ='80%' src ="/blog_photos/MPU/IVT_table.png"/><p>Table. IVT offset nxp i.mx6ull</p></div>

-3. IVT 表中的内容中有几个比较重要： (1).entry: 说明了可执行程序的起始地址(此地)。这个连接地址可以是imx6内部的RAM仅128KB(0X900000~0X91FFFF)，也可是外部的RAM（DDR 起始于 0X80000000，终止256MB:0X8FFFFFFF,512:0X9FFFFFFF）. (2).dcd的地址(DeviceConfig Data) dcd中存储了部分寄存器的初始化值如外设的时钟、初始化 DDR。
<div align=center><img width = '80%' height ='80%' src ="/blog_photos/MPU/IVT_content.png"/><p>Table. IVT content of nxp i.mx6ull</p></div>



### 裸机程序的初始化

####
vector table






