---
title: BIT eNSP 计网实验心得
published: 2026-04-26
pinned: true
description: 2025-2026春 BIT计科 计网乐学实验
tags: [Computer Network, eNSP]
category: Coursework
licenseName: "Unlicensed"
author: akanade
sourceLink: 
draft: false
date: 2026-04-26
image: "./cover.webp"
pubDate: 2026-04-26
permalink: 
---

实验列表：

> Lab-2 VLAN Configuration and Verification
>


实验环境：

> 系统：Windows11
>
> 软件：eNSP V100R003C00SPC100, VirtualBox-5.2.44-139111-Win(Win71011), Wireshark 4.6.4
> 


乐学eNSP实验总体难度很低，给的实验手册非常详细，按照步骤即可完成实验，主要在于体验计算机网络的各种特性，唯一的困难似乎是把英文读懂。以下是笔者认为略有阻碍的点：

### 1. eNSP中启动设备报错40

几乎所有人都会遇到的问题。原因是微软在之前的更新当中悄悄打开了"基于虚拟化的安全性"的开关，虽然Windows 功能开启和关闭 没有开启Hyper等功能。但运行msinfo查看系统信息可以看到倒数第七条就显示打开了。

解决办法：首先cmd管理员模式下运行 bcdedit /set hypervisorlaunchtype off ；然后gpedit.msc 计算机配置 - 管理模板 - 系统 - Device Guard 的“打开基于虚拟化的安全”策略，设为“已禁用”。最后重启一下就可以了。

所有实验报告都在GitHub仓库中：D

### 2. 关闭交换机的info输出

交换机会定时输出日志，对实验来说没用，反而会打断输入命令

```
undo info-center enable
```

### 3. eNSP中AR和PC等设备的配置保存问题

在eNSP中，点击工具栏的保存只会保存当前的拓扑结构，但并不会保存每个设备的配置（比如你给LSW接口配置的VLAN等），虽然可以在CLI中保存和加载配置，但是对于这些小实验来说不那么值当，所以还是推荐一口气做完一个Lab不要中断。

不过即使这样你依然可能被阴到。

例如Lab2会让你在2.1小节中配置一个按接口划分的VLAN，后续小节在此基础上扩充按MAC和IP划分的VLAN，但最后一节又会让你回到2.1的状态。此时你若直接按手册上写得加载在2.1时保存的拓扑就会丢失配置（手册甚至还贴心地在后面写到如果丢失接口配置就再重新配置一遍）。

又例如Lab3会让你在2.4小节的配置上继续实验。