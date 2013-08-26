---
layout: post
title: "LVM Extend Or Reduce"
description: "LVM 容量伸缩控制可以热调整，而这只依赖很简单的几个指令"
category: tech
tags: [tech, sa, debian, xen-tools, xen]
---
{% include JB/setup %}

整理自我自己的两篇旧博客，分别介绍 LVM 伸缩控制。

# 寄宿于LVM存储的xen虚拟机扩容

新主机中划分成一个xen虚拟机集群，全部部署在 lvm 存储上。昨天拖数据的时候发现文件服务器的空间规划比较紧张。于是准备给它扩容。

网上查了很多中文文档，不知道是因为太旧还是什么原因，没有一个靠谱的，操作复杂而且不安全。

最终在一篇英文文章中找到了一个办法，极其简单，并且验证确实可行。

我用来管理虚拟机的是debian的xen-tools，它自动的给基于lvm卷的xen虚拟机分配两个卷，一个disk，一个swap。接下来我们直接给disk扩容。

首先，在宿主机上给lvm扩容。为了安全起见，最好是先把要扩容的虚拟机停掉，实际上我操作的时候忘了先停机……（最好关，重要服务要先备份！）

lvextand，这个使用很简单。假设这个虚拟机的lvm卷是 /dev/stack/vm-disk

```
  lvextend --size +256G /dev/stack/vm-disk  
```

然后启动虚拟机，从ssh登录。
确认处于root身份（或者sudo也行），用fdisk　查看设备列表：

```
  fdisk -l  
```

你会看到若干存储设备，如果是跟我同样的环境，/dev/xvda2　就是我们刚刚操作的disk文件……（默认swap是1，disk是2）。此时我们会看到，parted或fdisk已经可以查看到设备的容量上调了，但是如果 df -h，看到的仍然是旧的容量。
然后执行

```
  resize2fs /dev/xvda2  
```

请耐心等待，命令完成后，扩容就成功了，此时df -h，会看到空间已经上调。不需要像那些中文文章所言的umount系统分区、init 2启动，修改分区表等危险操作！

# lvm 空间缩减操作

前几天学会了给lvm动态扩容，有次遇到缩容操作，也照方抓药，却惨遭失败。还好是开发机，默默重装。

问题在哪里呢？搜索了一些资料，在国外的一些社区提到：扩容时，先 lvextend 再 resize2fs ，缩减时先 resize2fs 再 lvreduce。

具体操作了一下，按这里的步骤，可以让lvm2+ext2正确缩减：

[http://www.microhowto.info/howto/reduce_the_size_of_an_ext2_ext3_or_ext4_filesystem.html]

具体步骤呢，以我的开发机为例，这里有一个 /dev/dumpling/storage 设备，挂在 /storage 上，它现在的容量是 8G，我希望它缩减到 4G。

首先卸载设备：

```
  umount /storage  
```

然后执行 fsck：

```
  fsck -f /dev/dumpling/storage  
```

再 resize2fs：

```
  resize2fs /dev/dumpling/storage 4G  
```

再 lvresize：

```
  lvresize --size 4G /dev/dumpling/storage  
```