---
layout: tech_post
title:  "Embedded linux system architecture"
date:   2020-05-25 11:51:36 -0300
catalogue: Mcu&Mpu
tags: Linux Uboot Dirver
description:  Kernel, Uboot, Rootfs(root file system) introduction 
---
* TOC
{:toc}

U-boot(bootloader), Linux Kernel and Rootsf are three key components of a embedded linux system.

<span style = "color:red;">Caution: The configurations and operations are mainly based on nxp i.mx series</span>

## 1. U-boot 
U-boot is the abbreviation and shorten of 'Universal Boot Loader', which support multiple architectures like ARM, MIPS, RISC-V and x86.

### 1.1 basic of uboot and flashing

The official Uboot source code can be found at [DENX-U-Boot](https://www.denx.de/wiki/U-Boot/SourceCode), also most of the SOCs manufactures will give their customized U-boot configuration, for example we can find most of the configurations of [NXP's Socs here](http://47.105.163.166/cyndent/uboot-imx).  

Before make an Uboot for one board some configurations should be done first, 
Using the following command makefile will create `.config` under the fold.

```c
make ARCH=arm clean
make ARCH=arm XXXX_defconfig
//make ARCH=arm mx6ull_14x14_evk_defconfig   ,   for freescale i.mx6u board SD card start
//make ARCH=arm mx6ull_14x14_evk_ emmc _defconfig   
```
Following command can make the corresponding U-boot file according to the `.config`
 
 ```
 make -j8 ARCH=arm CROSS_COMPILE=arm-none-eabi- `
```
We can see the U-boot as a big bare-metal program. So the way used to flash Uboot to SD card is same with bare-metal program. To flash the program some header information should be added to `U-boot.bin` file, which 
Here we use the `dd` command to flash.

According to the official image vector table of nxp IVT, we need to flash the U-boot to the address offset 1kBytes. 

`sudo dd if=u-boot.imx of=/dev/sdb bs=512 seek=2 conv=fsync`  

<div align=center><img width = '80%' height ='80%' src ="/blog_photos/MPU/IVT_table.png"/><p>Table. IVT of nxp i.mx6ull</p></div>


### 1.2 porting uboot to hardware

To adding support of our customized board, most of the time, we only need the modify the U-boot from soc manufacture. There are three important path we need modify(take imx6u as example, U-boot version>2018):

```
/configs/<board_name>
/include/configs/<board_name>
/arch/arm/cpu/armv7/mash-imx/mx6/Kconfig   //adding config TARGET and source "board/<vendor>/<board_name>
/board/<vendor>/<board_name>

```
The above configuration is the basic structure for a customized board. For the detailed configuration, I will write in [another blog](../../../2020/06/12/Uboot.html) about adding support of LCD, ethernet, and other devices.


### 2. Linux kernel


#### 2.1 Compile the official kernel  
The structure of Linux kernel is very similar with U-boot project. Also, we can use [nxp offical linux kernel](https://github.com/Freescale/linux-fslc) as our template to customize our kernel (git switch x.xx.x--imx:using the branch with --imx).

`make ARCH=arm clean `<br/>
`make ARCH=arm XXX_defconfig` (imx_v7_defconfig for nxp offical imx6)<br/>
`make menuconfig`(<span style="color:red;">optional</span> if dont need config just esc)<br/>
`make ARCH=arm -j10 CROSS_COMPILE= arm-linux-gnueabihf-`<br/> 


<span style="color:red;">Note:</span> Sometimes make cannot go through due to gcc version. In that case using older arm-linux-nueabihf- from source with `export PATH=/{your gcc address}/bin:$PATH` <br/> 


#### 2.2 Customize our linux kernel 


    To customize a linux kernel, two important file are needed:
    - xxx_defconfig   (in arch/arm/configs/)
    - xxx.dts (in arch/arm/boot/dts ) 

#### 



### 3. Rootfs 

rootfs can be understand as the container files  of linux environment, configuration, and programs in the mpu.    

#### 3.1 Roofts make

Rootfs can be made from various sources, busybox, buildroot, yocto, openembedded. 


#### 3.2 mount rootfs from nfs server
setenv bootargs 'console=ttymxc0,115200 nfsvers=4 root=/dev/nfs nfsroot=192.168.0.188:/home/jie/workdir/rootfs ip=192.168.0.16:192.168.0.188:192.168.0.1:255.255.255.0::eth1:off'

