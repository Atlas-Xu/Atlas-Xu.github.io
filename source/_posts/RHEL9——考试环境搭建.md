---
title: RHEL9——考试环境搭建
tags:
  - Linux
abbrlink: 6780e3f3
date: 2024-02-27 18:26:00
---

由于我本地的设备都开了Docker Desktop，因此在虚拟化方面和VMware有冲突。
想着家里还有服务器，那就把考试环境全部搭建在Proxmox Virtuia Environment上了。
<!-- more -->
虚拟机是由培训机构直接给出的考试环境虚拟机。

# PVE导入OVF
1. 这部分都是常规操作了，将ovf和vmdk上传到pve，导入ovf：`qm importovf 20240226001 RHEL9_Foundation_20230619.ovf HDD-Raid1-500G --format qcow2`

2. 将vmdk文件转为qcow2：`qemu-img convert -p -f vmdk -O qcow2 RHEL9_Foundation_20230619-disk1.vmdk RHEL9_Foundation_20230619-disk1.qcow2`

3. 重新挂载转换后的磁盘文件：`qm importdisk 105 RHEL9_Foundation_20230619-disk1.qcow2 HDD-Raid1-500G `

# 遇到的问题
## 引导问题
RHEL9的版本是需要UEFI引导的，在导入后一定得修改，不然就会卡在启动项中。如下图
![](https://data.xchub.cn/RHEL卡引导.png)

## 虚拟化后的指令集问题
RHEL9的“特性”之一——cpu支持的指令集版本，没错，现在指令集也有版本了（其实也不能说版本吧，更贴切应该说是指令集支持等级/程度），详细了解可以看 https://en.wikipedia.org/wiki/X86-64#Microarchitecture_levels
在修改完引导后，就会看到指令集不支持的情况，因为pve虚拟化后的设备都是比较古早的，因此将虚拟化直接改完host，即使用硬件CPU来支持就顺利进入。
![](https://data.xchub.cn/指令集不支持.png)

# 外部访问
简单配置路由器的端口转发和DDNS，DDNS可以详见家庭网络搭建的内容，后续即可用域名和端口号直接访问，公网上还是不开22端口了，太容易烂了，后续再上证书，随时随地的练习环境搞定。