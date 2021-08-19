---
layout: post
title: "Shapefile 格式解析"
subtitle: ""
excerpt: ""
date: 2021-07-07
author: "Civitasv"
catalog: true
header-style: text
mathjax: true
tags:
  - gis
  - shapefile
---

对于正数（00000001）原码来说，首位表示符号位，反码 补码都是本身
对于负数（10000001）原码来说，反码是对原码除了符号位之外作取反运算即（11111110），补码是对反码作+1 运算即（11111111）
当 byte 要转化为 int 的时候，高的 24 位必然会补 1，这样，其二进制补码其实已经不一致了，&0xff 可以将高的 24 位置为 0，低 8 位保持原样。这样做的目的就是为了保证二进制数据的一致性。
