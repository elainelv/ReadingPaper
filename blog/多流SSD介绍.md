---
title: "[Multi-stream]The Multi-streamed Solid-State Drive"
date: 2018-12-10
catalog: true
toc_nav_num: true
tags:
	- SSD
	- Multi-stream
categories: SSD
header-img: "/img/article_header/streamssd.jpg"

---

最先由三星公司提出的多流SSD技术，这篇主要介绍多流SSD
[原文链接](https://www.usenix.org/system/files/conference/hotstorage14/hotstorage14-paper-kang.pdf)
[PPT](http://www.usenix.org/sites/default/files/conference/protected-files/hotstorage14_slides_kang.pdf)
[作者报告视频](https://2459d6dc103cb5933875-c0245c5c937c5dedcca3f1764ecc9b2f.ssl.cf2.rackcdn.com/hotstorage14/kang.mp4)
<!--more-->

>最近在看Multi-Stream SSD方面的paper，比如上一篇博文中提到的FStream和vStream等等，了解了他们的ideas后，我们也有新的ideas。尝试着改代码！昨晚有了点思绪后，今天改了好多但是后来发现还是存在很多的问题，没法继续改下去了……
>明天去参加[2018中国存储与数据峰会](http://www.datastoragesummit.com/),有时间会专门写一篇博文和大家分享的！下面来一起来学习Multi-Stream SSD吧~是不是听着名字就感觉很先进啊^^

再添一篇介绍多流SSD的文章：[用多流写技术提高SSD的性能和寿命 ](http://www.sohu.com/a/111351776_165716)

### 研究背景
大家买电脑的时候是否会考虑买容量大价格便宜的HDD（机械硬盘）还是价格贵一些但是性能更好速度更快的SSD（固态硬盘）？
我们都知道机械硬盘即是传统普通硬盘，主要由：盘片，磁头，盘片转轴及控制电机，磁头控制器，数据转换器，接口，缓存等几个部分组成。上图！
![](/img/article/多流SSD介绍/5.jpg)
而固态硬盘用固态电子存储芯片阵列而制成的硬盘，由控制单元和存储单元（FLASH芯片、DRAM芯片）组成。上图！
![](/img/article/多流SSD介绍/7.jpg)
相比于HDD，SSD具有读写速度快、防震抗摔性、低功耗、无噪音、轻便等优点，但是其致命的缺点就是寿命限制，固态硬盘闪存的擦除次数限制问题，还有就是容量小价格高！

>PS：我买的电脑型号是T480，双盘（128GSSD+1THDD),电脑负载大的时候磁盘声音兹拉兹啦的响，虽说当时买电脑考虑到既要速度快又要容量高，因此就买了双盘，用着还不错，就是这个声音的bug一直很是难受了

#### SSD老化
`SSD老化`解释了为什么SSD性能可能会随着时间的推移而逐渐降低; 由于SSD充满了越来越多的数据并且这些数据呈碎片化,因此GC执行得更频繁。当使用“clean”NAND闪存时,老化效应开始显现。并且在这种情况下,FTL必须通过“erase”一些闪存块来主动恢复足够量的新容量,才能容纳新的写入数据。而进行erase操作之前通常先是执行花销比较高的GC,更糟糕的是,NAND块,一个擦除操作单元,在现代NAND闪存中相当大,其中有128 page或者更多。

当SSD充满越来越多的数据时,统计上,FTL需要在每次NAND闪存擦除操作之前复制更多有效的GC页面。这种现象类似于日志结构文件系统的“segment cleaning”,此现象已被充分研究。

![](/img/article/多流SSD介绍/1.png)

上图清楚地描述了SSD老化的影响：Cassandra吞吐量（在“正常”SSD的情况下）的性能严重下降，因为SSD中的数据集不断更新 - 高达~56％。

### 多流
在描述所提出的多流方法之前，让我们考虑揭示一些例子来解释为什么传统的写入模式优化（如仅附加日志记录）不能完全解决SSD老化问题。
![](/img/article/多流SSD介绍/2.png)
上图给出了示例，其中填充了两个NAND闪存块（块0和块1）并写入新数据以填充块2.在第一个示例（左）中，应用了顺序写入模式，并且 结果，一些数据在块0和1中变得无效。另一方面，在第二个例子中，应用了随机写入模式，使块0中的所有数据无效，但块1中没有。显然，未来的GC将更有效地进行
在此示例中，因为可以快速回收空的NAND闪存块（块0）而无需复制数据。这些示例表明SSD的GC头不仅取决于当前的写入模式，还取决于数据如何已经放置在SSD中

`SSD老化问题的核心是如何预测写入SSD的数据的寿命，以及如何确保具有类似寿命的数据被放置在同一个擦除单元中`。这项工作提出了多流，一个直接指导SSD内数据放置的接口，将这两个问题分开。我们认为主机系统应该（并且可以）为SSD提供有关数据生命周期的充分信息。然后，SSD负责将具有相似寿命的数据（由主机系统指示）放入相同的擦除单元中。

The Multi-stream SSD的设计引入了流的概念。Stream是SSD容量分配的抽象，它存储一组具有`相同生存期`的数据。实现多流接口的SSD允许主机系统打开或关闭流或者写入其中一个流。在写入数据之前，主机系统根据需要打开流（通过特殊的SSD命令）。

![](/img/article/多流SSD介绍/3.png)
主机系统和SSD共享每个打开流的`唯一流ID`，并且主机系统用适当的Stream ID增加每个写入。多流SSD严格分配物理容量，将数据放在一起，而不是混合来自不同流的数据。上图说明了如何实现这一目标。

"I want to say"
通过将具有不同生命周期的应用程序和系统数据映射到SSD流，可显著改善SSD吞吐量和延迟QoS
主要是优化了GC，极大地缩小了写放大WAF参数。

>先码到这，最近都会更新这方面的博文，有兴趣可以一起来研究交流哦！


