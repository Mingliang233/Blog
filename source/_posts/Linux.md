---
title: JVM
date: 2020/5/30 20:46:25
updated: 2020/7/27 21:23:25
categories:
- 后端工程师
tags:
- linux
---


用户态和内核态

首先虚拟内存是一个映射了物理内存（比如磁盘、RAM）的地址表。
操作系统又将虚拟内存被分为内核空间和用户空间。
分开主要是考虑安全性，不想让处于用户态的进程访问操作系统的虚拟内存。
用户态和内核态是处理器运行的两种模式。
应用程序在用户态下运行，多数驱动程序和核心系统组件在内核态下运行。
内核态拥有最高权限，可以访问任意的内存。用户态权力有限，只能访问有限的资源。
当执行程序的时候，一涉及到系统调用相关的代码，就会进行用户态到内核态的切换。

linux命令

整机 top
CPU vmstat
内存 free
硬盘 df
磁盘IO iostat
网络IO ifstat

https://blog.csdn.net/qq_38762479/article/details/89306984
https://blog.csdn.net/u012588160/article/details/100108895
https://blog.csdn.net/u010970712/article/details/80042731
https://docs.oracle.com/javase/specs/jvms/se14/html/index.html
https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20%E8%99%9A%E6%8B%9F%E6%9C%BA.md Cs-Notes
https://www.bilibili.com/read/cv5216534/?spm_id_from=333.788.b_636f6d6d656e74.7 尚硅谷路线图2020
https://www.cnblogs.com/onepixel/articles/7674659.html 排序算法 
https://computer.howstuffworks.com/ram.htm RAM工作原理