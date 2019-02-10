---
title: "[Multi-Stream]多流SSD的相关paper"
date: 2018-12-5
catalog: true
toc_nav_num: true
tags: 
    - Muti-stream SSD
    - File System
categories: SSD
header-img: "/img/article_header/multistreamssd.jpg"

---

1. [FStream: Managing Flash Streams in the File System](https://www.usenix.org/system/files/conference/fast18/fast18-rho.pdf)
[PPT](https://www.usenix.org/sites/default/files/conference/protected-files/fast18_slides_rho.pdf)
2. [High-Performance Transaction Processing in Journaling File Systems](https://www.usenix.org/system/files/conference/fast18/fast18-son.pdf)
[PPT](https://www.usenix.org/sites/default/files/conference/protected-files/fast18_slides_son.pdf)
3. [vStream Virtual Stream Management for Multi-streamed SSDs](https://www.usenix.org/system/files/conference/hotstorage18/hotstorage18-paper-yong.pdf)
<!--more-->

## 多流ssd笔记
对于GC回收的数据作为冷数据处理；对于重写的数据作为更热的数据处理

多流SSD的提出主要是为了降低GC的开销，将具有相同的stream ID的数据分组在同一闪存块中，从而降低GC成本并提高性能。

### 一.vStream
研究背景：
在商业SSD中，由于硬件设备资源的限制，仅支持少量的流。在这篇文章中提出了vstream, 通过对主机写请求的分配的流称之为虚拟流，由于硬件设备的物理流数量很少，通过计算每个vStream 的平均寿命和一个物理流的中值寿命的欧几里得距离，将虚拟流映射到距离比较近的物理流。
stream的概念，将具有相同生命周期的数据分配给相同的流ID，相同流ID的数据存于同一个闪存块中。这样，这些数据因为很有可能同时失效，降低了GC操作所带来的开销问题。值得我们注意的是，GC操作不仅降低了SSD的性能，还降低了其使用寿命，因为在SSd 中，每个闪存块的擦除次数是有限制的。

如何对stream进行分类？
这篇文章中介绍的算法是K-均值聚类算法，以对vStream的生命周期进行分类。

这种虚拟流的提出为主机系统提供了足够的流以充分利用多流SSD

### 二. fStream

为了解决SSd老化和GC开销问题而提出的。
随着SSD的使用，内部媒体碎片不断增加，这样GC操作就变得频繁起来。那么本文主要利用多流SSd技术，在文件系统层对写请求进行分类。

### 三. Journaling
由于日志文件的特殊性，我们在做GC操作时，可以考虑当前block的有效页中，哪些数据虽然还是有效的，但是，数据并没有那么重要，因此我们可以提前当作无效页一并擦除。

******

"I want to say"
多流SSD的使用极大地降低了GC带来的写操作，但是并行性方面并没有改善，可能还降低了，还在调研研究中。之后会持续更新博文进行这方面的说明。
