---
layout: post
title: "动态类型、静态类型、强类型、弱类型"
excerpt: "动态类型、静态类型、强类型、弱类型之间的差异"
date: 2021-08-19
author: "Civitasv"
catalog: true
header-style: text
tags:
  - programming
--- 

## 动态类型 vs 静态类型

动态类型语言和静态类型语言主要有两大差异：

- 第一，动态类型语言在**运行时**进行类型检查，静态类型语言在**编译时**进行类型检查；

  python 为动态类型语言，如下代码：

  ```python
  def runtimeerror():
      s = 'cat' - 1

  print("hi")
  ```

  在我们调用`runtimeerror`之前，代码不会报错，但一旦调用`runtimeerror`方法，将出现`TypeError`错误。

  java 为静态类型语言，如下代码：

  ```java
  public void test(){
      int x = 0;
      boolean b = x; // compile error!
  }
  ```

  该错误将在编译时出现。

- 第二，静态类型语言需要显式声明变量类型，动态类型语言则不需要。

  如下代码：

  ```java
  // java example
  int num;
  num = 5;
  ```

  ```python
  # python example
  num = 5;
  ```

  这两个例子均创建`num`变量，并指定变量值为 5。

## 强类型 vs 弱类型

强弱类型指你是否可以绕过类型系统，如 C 语言是典型的的弱类型，通过强制转换，任何指针类型都可以转换为其他指针类型，Java 则是强类型语言。

## 参考

- [Dynamic typing vs. static typing](https://docs.oracle.com/cd/E57471_01/bigData.100/extensions_bdd/src/cext_transform_typing.html)
