---
title: "[NVMe]读NVMe之DB"
date: 2018-12-27
catalog: true
toc_nav_num: true
tags:
	- SSD
	- NVMe
	- Doorbell Register
categories: NVMe
header-img: "/img/article_header/NVMe1.jpg"

---

继续上次NVMe的学习，在上次的博文中，我和大家分享了NVMe中的两大重要的角色SQ和CQ，那么在这次的博文中，我们要一起探讨的是第三个重要的角色DB，话不多说！直接进入主题吧~
<!--more-->

## 导语
>还没学习过NVMe之SQ与CQ的请戳[这里](https://www.elainelv.top/2018/12/18/读NVMe之SQ与CQ/#more),上次也提到了DB的功能，即`host发送命令时，不是直接往SSD中发送命令，而是把准备好的命令放到内存中的SQ，通过写SSD中的DB寄存器来通知SSD`,这里再提一哈，SQ与CQ都是循环队列。可以看出，DB在这里的角色就是一个负责传递信息的中介

## DB执行过程
DB：`Doorbell Register，位于SSD的控制器内部`
>在DB中记录了SQ的Head和Tail以及CQ的Head和Tail，那么host和SSD之间通信如何来更改这些值呢？

上图！(从[ssdfans](http://www.ssdfans.com/blog/2017/08/03/蛋蛋读nvme之二/)扣过来的)
![](/img/article/读NVMe之DB/1.png)

从图中我们可以看到，在SSD的控制器内，Doorbell_written写方分别是`Host`和`Controller`，`SQ1 Head`和`CQ Tail`由`Controller`控制，而`SQ1 Tail`和`CQ1 Head`由`Host`控制
1. 首先SQ1和CQ1均为空，Head和Tail均为0
2. 现在往SQ1中写入三条命令，那么host在往SQ1写入命令的同时，告知SSD Controller的`SQ1 Tail`，SQ1 Tail=3
3. 此时，SSD Controller知道SQ中有命令待处理，从SQ1中取出命令执行后（这里组要注意的是，SSD每次取多少条命令进行处理是不确定的），那么如果全部取出的话，`SQ1 Head`=0（SSD可根据本地存储的Tail值和当前取出的命令条数进行计算）
4. 当SSD执行完这三条命令后，将完成状态信息写入到内存的CQ1队列中，并更新本地的`CQ1 Tail`
5. Host接收到SSD发来的完成状态信息后，进行处理，待处理完这两条信息之后，告诉SSD，并更新`CQ1 Head`

**这里需要注意的是：**
host知道SQ1 Tail和CQ1 Head，但是不知道SQ1 Head和CQ1 Tail，那么这些信息是如何获取的呢？
我们可以想到的是，当SSD处理完命令发给host完成状态信息时，可以附带上SQ1 Head和CQ1 Tail
![](/img/article/读NVMe之DB/2.png)
如上图所示，SQ Head Pointer存的是SQ1 head，标记位P用来表示当前完成状态信息是否处理完，当从SSD返回给Host的信息中，标记位P的值为1，等到host处理完成后改为0，如下图所示
![](/img/article/读NVMe之DB/3.png)

## 总结一哈
- DB在SSD Controller端，是寄存器
- DB记录着SQ和CQ的Head和Tail
- 每个SQ或者CQ有两个DB: Head DB 和Tail DB
- Host只能写DB，不能读DB
- Host通过SSD往CQ中写入的命令完成状态获取Head或Tail

**参考资料：**
[蛋蛋读nvme之二](http://www.ssdfans.com/blog/2017/08/03/蛋蛋读nvme之二/)

## "I Want To Say"
>DB作为host和SSD间数据交互的中介，是一个寄存器，其实很多地方也会设置相应的缓存寄存器来存储数据。这里从名字取为Doorbell也是很通俗易懂了，相当于一个门铃嘛


