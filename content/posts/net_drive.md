+++
title = "网络设备驱动"
date = "2023-05-04T16:18:12+08:00"
author = "Civitasv"
authorTwitter = "" #do not include @
cover = ""
tags = ["驱动", "网络"]
keywords = ["", ""]
description = "对网络设备驱动的总结"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

网络设备驱动最主要的工作是：

1. 驱动网络设备（通常也叫网卡）将数据发送出去
2. 将网络设备收到的数据往上层递交

在这里，网络设备驱动担当了承上启下的作用，上层的网络协议层不必关心底层的硬件细节信息。

数据接收或发送时，网络设备驱动一般通过中断机制进行相应的处理。

## 什么是网卡

网卡最主要的工作是：

1. 发送数据时，将数据封装为帧，并通过光模块将数据发送到网络
2. 接收数据时，将帧解包为网络层数据，使用 DMA 将数据搬运到主机

## checksum

目的是验证报文在网络传输过程中的正确性。