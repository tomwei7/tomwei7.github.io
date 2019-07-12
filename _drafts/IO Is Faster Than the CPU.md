---
layout: post
title:  "[译]I/O Is Faster Than the CPU – Let’s Partition Resources and Eliminate (Most) OS Abstractions"
date:   2019-07-03 22:48:58 +0800
categories: translate
---

### 摘要

因为快速的可编程 NIC (网络适配器) 和接近 DRAM 速度的 NVME 闪存的出现，服务器的 I/O 速度正在变得越来越快。但是 CPU 单线程的速度停滞不前。当使用的接口抽象是建立在围绕着 I/O 速度很慢的基础上时应用就无法充分的发挥现代硬件的优势。因此我们提出了名为 Parakernal 的操作系统结构，它能够消除大多数操作系统抽象的同时并为应用程序提供能充分利用底层硬件潜力的接口。(Parakernel 通过安全地分区资源并仅复用那些未分区的资源来促进应用程序级并行性。)

> The parakernel facilitates application-level parallelism by securely partitioning the resources and multiplexing only those resources that are not partitioned.

### 参考

[I/O Is Faster Than the CPU - Pekka Enberg](https://penberg.org/parakernel-hotos19.pdf)
