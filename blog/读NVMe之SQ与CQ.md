---
title: "[NVMe]读NVMe之SQ与CQ"
date: 2018-12-18
catalog: true
toc_nav_num: true
tags:
	- SSD
	- NVMe
categories: NVMe
header-img: "/img/article_header/NVMe2.jpg"

---

对[NVMe 1.3c](https://nvmexpress.org/wp-content/uploads/NVM-Express-1_3c-2018.05.24-Ratified.pdf)的解读，本篇专门讲Submission Queue和Completion Queue，NVMe于2013年提出，是host与SSD之间通信的协议，属于应用层，而与之配合使用的PCIe属于下层，这两者均是为SSD而生的，在之前的HDD时代用的是AHCI和SATA的协议。当然在没有NVMe出现之前，SSD走的也是AHCI和SATA，SATA读取性能最高不会超过600MB/s，太慢了……显然不足以满足目前SSD的高性能要求，SSD需要PCIe，同样需要NVMe！

<!--more-->

## 导语
>[NVMe specification](https://nvmexpress.org/resources/specifications/)
最近一段时间会把上面的文档大致读完，将比较有意思的内容我会连载着写写博文，记录读文档过程中遇到的重要内容和问题，话不多说，一起来揭开 NVMe的神秘面纱哈~

NVM Express是主机和非易失性内存子系统间进行通信的interface，此interface针对企业和客户端固态驱动器进行优化，通常作为寄存器级interface附加到PCIExpress接口。
先来看两张图，主机和控制器之间进行通信通过SQ和CQ，其中NVMe命令分为`Admin Command`和`I/O Command`
`Admin Command：`用以host管理和控制SSD
`I/O Command：`用以host和SSD之间数据的传输
![](/img/article/读NVMe之SQ与CQ/1.png)

![](/img/article/读NVMe之SQ与CQ/2.png)

正如上图所示，系统中只有一对Admin SQ/CQ,而I/O SQ/CQ却可以有很多，多达65535(64K减去一个SQ/CQ)。而实际系统中用多少个SQ，取决于系统配置和性能需求，`可灵活设置I/O SQ个数`。

作为队列，每个SQ和CQ都有一定的深度。对Admin SQ/CQ来说，其深度可以是2-4096（4k）；对I/O SQ/CQ,深度可以是2-65536(64k)。`队列深度也是可以配置的。`

SQ/CQ的个数可以自己配置，队列深度也可以自己配置，NVMe也太牛了吧！

>下面具体来讲解SQ和CQ以及我在阅读文档的过程中遇到的问题，相信大家看完应该会对两个概念有一定的了解！

## Submission Queue（SQ） 和 Completion Queue（CQ）
### SQ? and CQ?
先来看一张图（从[ssdfans](http://www.ssdfans.com/)上扣下来的）
![](/img/article/读NVMe之SQ与CQ/3.png)
`SQ：用来存放命令`
`CQ：用来存放提交后的命令经SSD处理后的完成信息`
两者均是存放在`memory`里面，均为**循环队列**（这里关于循环队列具备的一些特性就不详细解释了）
注：SQ和CQ是成对出现的；SQ和CQ的关系可以是一对一，也可以是多对一；SQ是在CQ创建之后才创建的

红圈圈圈起来的是`NVMe Subsystem`，包括`controller`、`NVM storage`、`Interface between them`。其实就是SSD啦。它通过PCIe和Root Complex(RC)连接，当然SSD是不能直接和CPU通信的。

`DB又是用来干嘛的呢？`
host发送命令时，不是直接往SSD中发送命令，而是把准备好的命令放到内存中的SQ，通过写SSD中的DB寄存器来通知SSD。

### 命令执行过程
下面着重讲解通过SQ和CQ之间的指令执行过程，如下图所示，分为8步(图片来自P210)：
![](/img/article/读NVMe之SQ与CQ/4.png)

1. host将写命令放入SQ的空闲槽中
2. SQ的尾指针更新，host通过SQ尾指针来更新DB寄存器的值，这向控制器发送指示说已经提交了一个新命令待处理（host写DB，通知SSD取指令）
3. SSD收到提示通知，从SQ中取指令；（若是多个控制器的结构，那么需要仲裁哪个控制器去执行）
4. SSD执行指令
5. 指令执行完成后，SSD往相关的CQ中写指令执行结果；（每个completion queue entry都有与前一个相反的阶段标记，以向主机指示该完成队列条目是一个new entry）
6. SSD发送信息通知host指令完成（这一步可能要写很多条指令的完成信息）
7. 收到信息后，host处理CQ中新的信息，查看指令完成状态（当然这一步可能出现错误，那么host继续处理CQ直到相位标记的值反转）
8. host处理完CQ指令执行结果，通过DB回复SSD，并把当前的Completion Entry释放掉。（主机写Completion Queue Head Doorbell register 以指示已经执行完CQ entry，在更新相关的Completion Queue Head Doorbell register之前，主机可能消耗许多entry；也就是说第8步可能是前面1~7步执行多次后才执行一次）

>这里值得注意的是，每个SQ中放入的命令无论是Admin还是I/O命令，每个命令大小都是64字节；每个CQ放入的是命令的完成状态，大小为16字节

## 总结一哈
- SQ用以host发命令，CQ用以SSD回命令表示命令完成，返回的是完成信息
- SQ/CQ可以在host的内存中，也可以在SSD中，但一般在host内存中（上述讲的都是以host中为例的）
- 两种类型的SQ和CQ：Admin和I/O，前者发送Admin命令，后者发送I/O命令
- 系统中只能有一对Admin和I/O，但可以有很多对I/O SQ/CQ
- I/O SQ与CQ可以是一对一或者一对多的关系
- I/O SQ 可以赋予不同的优先级(比如P77一张图)
![](/img/article/读NVMe之SQ与CQ/6.png)

- I/O SQ/CQ的广度和深度都可以灵活配置
- 每条命令（SQ中）大小64字节，而完成状态的命令（CQ中）16字节

**参考资料：**
[蛋蛋读NVMe之一](http://www.ssdfans.com/blog/2017/08/03/%E8%9B%8B%E8%9B%8B%E8%AF%BBnvme%E4%B9%8B%E4%B8%80/)
[蛋蛋读NVMe之二](http://www.ssdfans.com/blog/2017/08/03/%E8%9B%8B%E8%9B%8B%E8%AF%BBnvme%E4%B9%8B%E4%BA%8C/)

## "I Want To Say"
NVMe整本书浏览了一遍，感觉可以研究一段时间了，尽量每天都写东西吧，感觉有很多可以写的。当然这个协议规范里面很多都是讲内部指令呀，队列呀，内存呀怎么去定义的，还包括各种命令的使用……等等。反正就是很复杂啦~今天就码到这了，明天继续哦！








