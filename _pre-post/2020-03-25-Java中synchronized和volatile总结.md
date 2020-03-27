---
layout:     post
title:      Java synchronized 和 volatile 总结
subtitle:   Java中关键字synchronized与volatile相关原理知识点，分析入门
date:       2020-03-25
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - jol
    - 锁机制
---

> 正所谓前人栽树，后人乘凉。
> 
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

# 前言
网上有许多的文章对这两个知识点做了详细的叙述剖析，可能由于平台、JDK版本不尽相同，导致文章参差不齐，难以构成统一的知识体系。  
别人的知识终究不是自己的，自己再来总结一次。

# synchronized 简述

# synchronized 分析

- 1、对象创建后默认无锁状态，当线程访问时，升级为偏向锁；
- 2、若在同一线程中对象进入同步块前访问对象的默认hashCode方法，即Object类的HashCode方法，则偏向锁会被释放，进入同步块时升级为轻量级锁;
- 3、从2中的同步块退出时，轻量级锁会被释放，回到无锁状态；
- 4、1,2,3的原因是对象默认hashCode方法调用后，hashCode会存到mark word中，会占用偏向锁的ThreadId的空间。所以`若调用了默认的hashCode，对象将无法使用偏向锁`。

# volatile 简述

# volatile 分析