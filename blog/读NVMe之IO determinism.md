---
title: "[NVMe]读NVMe之I/O Determinism"
date: 2019-1-14
catalog: true
toc_nav_num: true
tags:
    - NVMe
    - I/O determinism
categories: NVMe
header-img: "/img/article_header/NVMe4.jpg"

---

目前NVMe官网上最新的一个版本是1.3c，也就是我正在看的这个版本，发布于2018年5月24日。而2019年，NVMe1.4版本将是NVMe标准化组织工作重点，此次更新的重点之一就是I/O Determinism——是一把提高QoS的利器！

<!--more-->

**NVMe协议族的路线图:**

![](/img/article/读NVMe之IOdeterminism/0.png)

<font color=#FF8C00>区分multi-stream与I/O determinism？</font>
multi-stream： 为了改善GC问题，使得具有相同生存周期的数据放入同个地方，那么在垃圾回收操作时，尽可能的减少额外的写操作。
I/O determinism：用户需要高速、可靠、稳定的I/O性能
- **I/O延迟问题? 考虑如何做并行化（多并行，少串行）**
什么意思呢？我们知道，I/O延时导致的I/O不稳定，波动严重，这体现在很多方面，比如手机使用时，突然的卡顿。那么试想着把各个应用分开，各自独立的使用各自的资源是否会解决这个问题。我们将整块的SSD划分为多个逻辑单元称之为`NVM set`，每个NVM Set可以包含1到多个Channel和Die，不同的NVM Set的擦除、读写都是相互独立的，在不同的逻辑单元`并行读和并行写`。做并行的操作减少串行的工作，而且应用之间不会互相干扰，性能和延迟也可以得到更好的保障
![](/img/article/读NVMe之IOdeterminism/1bb.png)
IO Determinism应用前后整个应用访问盘的变化(来自于Facebook在FMS2018上的报告)：
![](/img/article/读NVMe之IOdeterminism/9b.png)

>这里需要注意的是，NS和SET没有本质的关系，一个set对应的NS可以是一个也可以是多个
你可能会考虑这样一个问题：NS和set不都是划分SSD的空间，为了分开使用吗?这两者是不同的，set只是对SSD的空间简单的划分开，使得上层在使用的时候可以以set为一个单位。那么NS呢？它有name 和ID，每个控制器可以有多个NS，也可以共享NS。这两者的概念还是不同的。

Facebook在FMS2018上发布了关于NVMe SSD实现I/O Determinism详细的测试结果，如下图：
![](/img/article/读NVMe之IOdeterminism/10.png)
从上图Facebook对IO Determinism的测试结果可以看出，读延迟QoS在IO Determinism应用后有了8倍的提升

- **I/O优化级问题？设置I/O标签的功能**
我认为，设置这个优先级问题自然是为了改善用户的体验感，提高服务质量。例如：用于服务等级Qos的管理，利用附加在每一笔I/O的Qos等级标签，让SSD控制器优先处理QoS等级高的I/O。因为在所有的IO中，总有那些重要的紧急的处理，也有那些无关紧要的处理，我们需要保证的是那些紧急的任务尽快的被服务处理。

**SSD数据布局结构的影响:**
以4 Namespace, 8 Channels的SSD系统为例，传统的data布局如下图：最简单的结构配置，数据均匀分布在所有的die。但是这个布局的缺点就是会有IO冲突造成的延迟
![](/img/article/读NVMe之IOdeterminism/7.png)


**基于I/O determinism功能, 引入三种逻辑单元结构的数据布局：**
1. 垂直逻辑单元((Vertical Sets)：
![](/img/article/读NVMe之IOdeterminism/6.png)

2. 水平逻辑单元(Horizontal Sets):
![](/img/article/读NVMe之IOdeterminism/5.png)

3. 混合型逻辑单元(Mixed Sets):
![](/img/article/读NVMe之IOdeterminism/4.png)

 **IO Determinism中的时间窗**
![](/img/article/读NVMe之IOdeterminism/1.png)

NVMe在SSD里面设置了一个时间窗，这个时间窗是稳定时延的模式，即<font color=ff8c00>Determinism Window</font>，当SSD盘需要垃圾回收、Wear-leveling维持操作时切换到非稳定模式，这个阶段IO性能是不稳定的，即<font color=ff8c00>Non-Determinism Window</font>, 也可以称作Maintainance Window
![](/img/article/读NVMe之IOdeterminism/2.png)

如果用两块SSD合作，则在任何一个时间点，至少会有一个SSD处在Determinism Window，为系统提供稳定的IO性能，如下图：
![](/img/article/读NVMe之IOdeterminism/3.png)

**参考资料:**
[1].[Facebook:Enabling NVMe® I/O Determinism @ Scale](https://www.flashmemorysummit.com/English/Collaterals/Proceedings/2018/20180807_INVT-102A-1_Petersen.pdf)
[2].[Improving NVMe SSD I/O Determinism with PCIe Virtual Channel](https://ieeexplore.ieee.org/document/8094836/)
[3].[三星公司I/O Determinism and Its Impact on Data Centers and Hyperscale Applications](https://www.flashmemorysummit.com/English/Collaterals/Proceedings/2017/20170808_FB11_Martin.pdf)

**"I want to say"**

>不管是之前介绍的Multi-Stream还是本篇介绍的I/O Determinism基本都是NVME协议针对数据中心提供的新功能
>那么这个I/O determinism关键目标就在于提高稳定可靠的服务质量，解决所谓的I/O延迟问题，而我们需要清楚的是I/O延迟问题的产生是由于各应用间共享资源带来的竞争与等待，思路很简单，设置优先级，资源隔离，使得各个应用独立的使用各自分得的SSD资源。可能你会想将一块原本可以共享资源的SSD盘分成单个小盘，会不会资源利用率不高呢？不会的，现在的SSD容量做得越来越大，分成单个盘之后可以满足各应用的需求，而且，可以将那些互相合得来的应用依然共享同一个NVM set。这样一来，大大的提高的服务质量，提高了应用的并发程度（因为不同set中的应用互不干扰）降低延迟的同时也提高了资源的利用率
本部分内容还在学习中，目前整理了这些，部分来源于网上资料搜集学习整理，也含有个人自己的看法，如有不同看法的可以互相学习哦~
