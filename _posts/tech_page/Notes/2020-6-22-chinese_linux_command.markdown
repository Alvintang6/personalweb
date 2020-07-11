---
layout: tech_post
title:  " linux command "
date:   2020-6-23 11:51:36 -0300
catalogue: Notes
tags: Notes 
description: 关于liunx command的一些笔记
permalink: /chinese/linux_command.html
---
## 目录
* TOC
{:toc}

### linux 配置网络
#### 静态ip设定即时生效
 `ifconfig eth0 192.168.1.5 netmask 255.255.255.0`
#### 静态ip设定config文件 
- (1)   vi /etc/network/interfaces
- (2)  auto eth0  
	/#iface eth0 inet dhcp   
	iface eth1 inet dhcp 

	iface eth0 inet static  
	address 192.168.0.100  
	netmask 255.255.255.0  
	gateway 192.168.0.1  
	broadcast 192.168.0.255  

	#Wireless interfaces  
	iface wlan0 inet dhcp  
        wireless_mode managed  
        wireless_essid any  
        wpa-driver wext  
        wpa-conf /etc/wpa_supplicant.conf  
- (3) restart network card: /etc/init.d/networking restart
- (4 optional) add /etc/init.d/networking restart  into /etc/rcN.d/<SN_script.sh> initial script 


### linux 文件操作


- 从sd拷贝rootfs 
sudo mount /dev/sdb2 /mnt
sudo cp -r /mnt/* /<your home path>

- 拷贝rootfs到sd卡
sudo mount /dev/<your_sd_blk> /mnt
sudo mkfs.ext4 /dev/<your_sd_blk>
sudo cp -r /rootfs/* /mnt


### linux 查询操作
