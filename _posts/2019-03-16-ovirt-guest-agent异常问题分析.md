---
layout:     post
title:      ovirt-guest-agent异常问题分析
subtitle:   
date:       2019-03-16
author:     Kaerser
header-img: img/bg-01.jpg
catalog: true
tags:
    - ovirt
---

在当前新版本(4.1以上)安装node节点(操作系统版本CentOS7.5+)时，安装完之后发现集群中虚拟机无法迁移，并且ovirt-guest-agent状态异常

![异常截图](img/screenshot/ovirt-guest-agent-cpu&mem-err.png)

> 从以上截图提示虚拟机ovirt guest agent未安装，同时无法获取到虚拟机IP和主机名信息，包括实时的CPU和内存信息；另外集群中的虚拟机在线迁移功能一直报错；

**通过分析发现问题的原因是node节点上安装的libvirt版本升级到*4.X*以上，信息如下：**
![libvirt异常版本](img/screenshot/ovirt-libvirt-4.5.0.png)

> 正常libvirt版本应该为*3.X*，通过重新安装操作系统(必须重新安装操作系统)，并将libvirt的版本指定为3.X版本，才能解决ovirt-guest-agent异常和虚拟机迁移失败的问题

![libvirt正常版本](img/screenshot/ovirt-libvirt-3.9.0.png)

在完成libvirt版本安装为3.X之后查看虚拟机ovirt guest agent状态，均恢复正常：
![正常截图](img/screenshot/ovirt-guest-agent-cpu&mem-active.png)

**问题分析：**

1. CentOS默认yum源指定的base源和update源在CentOS7.6发布之后，默认指向了CentOS7.6的源，而通过查找CentOS7.6的镜像中libvirt的版本已经升级至4.5.0，在安装ovirt node节点时默认将libvirt版本升级至4.5.0，这是导致出现以上问题的根本原因；
2. 解决以上问题需要修改base源和update固定写至CentOS7.5，分别为：[base](http://vault.centos.org/centos/7.5.1804/os/x86_64/)，[update](http://vault.centos.org/centos/7.5.1804/updates/x86_64/)

**题外话：**

*造成libvirt 4.X版本出现问题的原因目前还未找到官方的解释，有关这方面的问题信息，大家可以在评论里联系我，谢谢...*

