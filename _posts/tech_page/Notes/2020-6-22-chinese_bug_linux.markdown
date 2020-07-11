---
layout: tech_post
title:  " linux system bug notes"
date:   2020-6-23 11:51:36 -0300
catalogue: 中文 Notes
tags: Notes 
description: 关于liunx 中遇到的问题的一些笔记
permalink: /chinese/linux_bug.html
---
## 目录
* TOC
{:toc}

### Can't find a valid FAT filesystem

启动时出现Can't find a valid FAT filesystem且只能进入紧急emergency mode.
可能是linux rootfs所在磁盘缺少FAT分区或者，1号内核分区格式不兼容。
- 缺少分区：
	sudo fdisk /dev/<your disk> n
	......
	w
	sudo mkfs.vfat -F 32 /dev/<your disk>
- 分区格式不匹配：
	sudo mkfs.vfat -F 32 /dev/<your disk>

### python3-rosdep-modules; python3-rosdistro-modules

The following packages have unmet dependencies:  
 python3-rosdep-modules : Depends: python3-rospkg-modules (>= 1.1.10) but it is not going to be installed  
                          Depends: python3-catkin-pkg-modules (>= 0.4.0) but it is not going to be installed  
                          Depends: python3-rosdistro-modules (>= 0.7.5) but it is not going to be installed  
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).  

Method 1:  
`sudo apt --fix-broken install`  

Method 2:
```shell
sudo mv /var/lib/dpkg/info /var/lib/dpkg/info.bk
sudo mkdir /var/lib/dpkg/info
sudo apt-get update
sudo apt-get install -f
```

Method 3(suggest):
Using aptitude managment software

sudo apt-get install aptitude
sudo aptitude install <confilct package>
