---
title: Arm64栈对齐规则
date: 2022-08-26 19:09:25
categories: compiler
tags: [arm64，栈对齐]
---

最近在bolt使能过程中，遇到一个SIGBUS的问题。在调用栈中，最后一层栈的报错指令是一条`stp x0, x1, [sp, #-16]!`，该指令是插桩过程添加的指令。一般来说，在函数入口处，都会通过指令`stp fp, lr, [sp, #-16]`保存fp和lr。理论上，两条指令并无本质区别。

报错的信息SIGBUS看起来又是一种对齐问题。进一步分析发现，在报错的位置，SP的值不是16byte对齐的。arm64平台上，通过SP访问栈，要求SP的值务必是16byte对齐。而出问题的函数，caller和callee有自己的约定，caller保存lr，并更新SP，callee负责保存fp。在callee函数中，不直接通过pre-index的方式存栈，而是使用`sub, sp, sp, #504` 的方式先开栈，这个过程同时保证了SP满足对齐要求。然后通过`str fp, [sp, #496]` `stp...`等指令将寄存器入栈。

可以写一个小例子说明上述问题,在汇编中写两条指令：
```assembly
str x0, [sp, #-8]!
str x1, [sp, #-8]!
```
会发现第二条指令处报SIGBUS error。