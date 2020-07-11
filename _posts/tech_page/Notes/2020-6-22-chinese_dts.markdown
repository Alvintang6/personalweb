---
layout: tech_post
title:  " 关于Device tree的一些笔记"
date:   2020-6-23 11:51:36 -0300
catalogue: Notes
tags: 中文 Notes 
description: 关于Device tree的一些笔记
permalink: /chinese/dts.html
---
## 目录
* TOC
{:toc}

###  设备树的主要作用
（在早期的Linux系统中，使用的是硬件描述文件的形式来实现该功能的。每一个具体的硬件平台都会在Linux系统源码包的arch/arm/mach-xxx/。 随着新的硬件平台不断产生，为了支持这些硬件平台，Linux系统中会增加越来越多的板级描述文件，从而导致系统中的冗杂文件越来越多。）


- 设备树是用来描述一款嵌入式设备的板级信息的（包括接口，地址，寄存器，部分自定义信息）。事实上设备树仅包含硬件信息，设备树做到了将硬件与软件信息解耦的作用。在开发中工程师仅需要着重开发软件驱动即可。
- 设备树与C语言类似支持include，实际上最基础的SOC信息会被芯片厂商写入xxx.dtbi文件中，使用时可直接基于dtbi文件写.dtb文件。


### 设备树与内核以及linux的关联

- 设备树中的"/"{ } 代表其为根节点， 所有根节点下的主节点均可在/proc/device-tree下查看。
- 在linux系统库中提供了一系列的of函数用于获取设备树中的信息
