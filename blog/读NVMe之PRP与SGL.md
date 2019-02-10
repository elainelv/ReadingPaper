---
title: "[NVMe]读NVMe之PRP与SGL"
date: 2019-1-12
catalog: true
toc_nav_num: true
tags:
	- SSD
	- NVMe
categories: NVMe
header-img: "/img/article_header/NVMe3.jpg"

---

>之前两篇博文介绍了SQ、CQ、DB三大命令传输时的重要数据结构。那么本篇博文和大家分享的是两大记录数据在内存位置的方式：PRP和SGL，在对SSD进行读写操作时，需要告诉SSD当前数据在内存的位置。我们知道NVMe1.3 版本中添加了新的功能，在了解了这些基础的内容之后将和大家一起分享目前研究的热点Multi-Stream以及I/O Determinism

<!--more-->

## 相关介绍
在讲正式内容之前，我们先来看下这张熟悉的图：
![](/img/article/读NVMe之PRP与SGL/11b.png)

PS：提醒大家注意一哈，NVMe Subsystem的概念，它包括一个或多个控制器、零或多个namespace、一个或多个端口、非易失性存储器存储介质以及控制器和非易失性存储器之间的接口。
___

**关于写数据**
Host如果想往SSD上写入用户数据，需要告诉SSD写入什么数据，写入多少数据，以及数据源在内存中的什么位置，这些信息包含在Host向SSD发送的Write命令中。每笔用户数据对应着一个叫做 **LBA(Logical Block Address)** 的东西，Write命令通过指定LBA来告诉SSD写入的是什么数据。对NVMe/PCIe来说，SSD收到Write命令后，`通过PCIe去Host的内存数据所在位置读取数据`，然后把这些数据写入到闪存中，同时得到LBA与闪存位置的映射关系

**关于读数据**
Host如果想读取SSD上的用户数据，同样需要告诉SSD需要什么数据，需要多少数据，以及数据最后需要放到Host内存的哪个位置上去，这些信息包含在Host向SSD发送的Read命令中。SSD根据LBA，查找映射表，找到对应闪存物理位置，然后读取闪存获得数据。数据从闪存读上来以后，对NVMe/PCIe来说，SSD会`通过PCIe把数据写入到Host指定的内存中`。这样就完成了Host对SSD的读访问

在上面的描述中，大家有没有注意到一个问题，那就是Host在与SSD的数据传输过程中，Host是被动的一方，SSD是主动的一方。你Host需要数据，是我SSD主动把数据写入到你的内存中；你Host写数据，同样是我SSD主动去你Host的内存中取数据，然后写入到闪存。SSD跟快递小哥一样辛劳，不仅送货上门，还上门取件

无论送货上门，还是上门取件，你都需要告诉快递小哥你的地址，不然茫茫人海，快递小哥怎么就能找到你呢？同样的，Host你不亲自传输数据，那总该告诉我SSD去你内存中什么地方取用户数据，或者要把数据写入到你内存中的什么位置。你在告诉快递小哥送货地址或者取件地址时，会说XX路XX号XX弄XX楼XX室，也可能会说XX小区XX楼XX室，anyway，快递小哥能找到就行

Host也有两种方式来告诉SSD数据所在内存位置，一是`PRP (Physical Region Page, 不是P2P!)`，二是`SGL (Scatter/Gather List)`。不过，后者感觉不怎么友善，因为怎么听起来都像”死过来”（SGL）。当然了，也可能是我误会了，人家只是在说”送过来”
___


## PRP

`PRP（physical region page）是一个指向物理内存页的指针，用于在控制器和内存之间数据传输的一种机制`。为了确保数据在控制器和主机之间高效传输，NVMe把Host的内存划分为一个一个页（Page），物理内存页的大小和偏移字段的大小由CC.MPS中的主机软件配置，大小可以是4KB,8KB,16KB … 128MB

![](/img/article/读NVMe之PRP与SGL/22b.png)

PRP Entry本质就是一个64位(63:00)内存物理地址，只不过把这个物理地址分成两部分：`页起始地址`和`页内偏移`。如果内存页大小是4KB(2的12次方），那么11:00是偏移地址（页内偏移大小必定是0~整个页的大小）；如果内存页大小是8KB（2的13次方），那么12:00是偏移地址。如果这个entry不是命令中第一个PRP entry或者不是一个PRP list指针，那么偏移量必须是0h；也就是说，偏移量应该是双字对齐(物理地址只能四字节对齐访问)，那么很明显最后两位1:0是0

>注意，控制器不需要检查1：0是否被清除为00b。 如果1：0未被清除为00b，则控制器可以报告PRP偏移的错误无效。 如果控制器未报告PRP偏移无效的错误，则控制器应将1：0位清除当作为00b一样运行

![](/img/article/读NVMe之PRP与SGL/33b.png)

从上述可以看出，一个PRP Entry描述的是一段连续的物理内存（从起始地址开始，长度为偏移量大小的一段内存区域开始）。

思考下，如果需要描述若干个不连续的物理内存呢？那就使用若干个PRP Entry呗！那么，把若干个PRP Entry链接起来，就成了PRP List。在这里你可能会想到操作系统学习的时候，用到的文件索引

![](/img/article/读NVMe之PRP与SGL/44b.png)

是的，正如你所见，`PRP List中的每个PRP Entry的偏移量都必须是0`，PRP List中的每个PRP Entry都描述一个物理页。它们不允许有相同的物理页，不然SSD往同一个物理页写入几次的数据，导致先写入的数据被覆盖。那么偏移地址为0可以保证每次读取当前起始地址开始的一个物理页大小。每个NVMe命令中有两个域：PRP1和PRP2，Host就是通过这两个域告诉SSD数据在内存中的位置或者数据需要写入的地址

![](/img/article/读NVMe之PRP与SGL/55b.png)

如果有多个PRP，那么这个PRP 的最后一个entry指向下一个PRP list第一个entry的起始地址。PRP1和PRP2有可能指向数据所在位置，也可能指向PRP List。类似C语言中的指针概念，PRP1和PRP2可能是指针，也可能是指针的指针，还有可能是指针的指针的指针。别管你包的有多严实，根据不同的命令，SSD总能一层一层的剥下包装，找到数据在内存的真正物理地址下面是一个PRP1指向PRP List的示例：
![](/img/article/读NVMe之PRP与SGL/66b.png)

PRP1指向一个PRP List，PRP List位于Page 200，页内偏移50的位置。SSD确定PRP1是个指向PRP List的指针后，就会去Host内存中（Page 200，Offset 50）把PRP List取过来。获得PRP List后，就获得数据的真正物理地址，SSD然后就会往这些物理地址读出或者写入数据

对Admin命令来说，它只用PRP告诉SSD内存物理地址；对I/O 命令来说，除了用PRP，Host还可以用SGL的方式来告诉SSD数据在内存中写入或者读取的物理地址。
![](/img/article/读NVMe之PRP与SGL/77b.png)

Host在命令中会告诉SSD采用何种方式。具体来说，如果命令当中DW0[15：14]是0，就是PRP的方式，否则就是SGL的方式。

## SGL

**SGL是什么？**
SGL是一个数据结构，用以描述一段数据buffer，这个数据buffer可以是数据源所在的空间，也可以是数据目标空间。SGL(Scatter Gather List)首先是个List，是个链表，由一个或者多个SGL Segment组成，而每个SGL Segment又由一个或者多个SGL Descriptor组成。只有SGL segment中的最后一个descriptor才可能是一个SGL segment descriptor或一个SGL Last Segment descriptor

最后一个SGL segment可能不含有SGL segment descriptor或者SGL Last Segment descriptor

`SGL Descriptor是SGL最基本的单元，它描述了一段连续的物理内存空间：起始地址+空间大小。`每个SGL Descriptor大小是`16字节`。

一块内存空间，可以用来放用户数据，也可以用来放SGL Segment，根据这段空间的不同用途，SGL Descriptor也分几种类型

![](/img/article/读NVMe之PRP与SGL/88b.png)

有4种SGL Descriptor:
(1) 一种是`Data Block`，这个好理解，就是描述的这段空间是用户数据空间；

(2)一种是`Segment描述符`。SGL不是由SGL Segment组成的链表吗？既然是链表，前面一个Segment就需要有个指针指向下一个Segment，这个指针就是SGL Segment描述符，它描述的是它下个Segment所在的空间。

(3) 特别地，对链表当中倒数第二个Segment，它的SGL Segment描述符我们把它叫做`SGL Last Segment描述符`。它本质还是SGL Segment描述符，描述的还是SGL Segment所在的空间。

>为什么需要把倒数第二个SGL Segment描述符单独的定义成一种类型呢？
我认为是让SSD在解析SGL的时候，碰到SGL Last Segment描述符，就知道链表快到头了，后面只有一个Segement了。

(4) 那么，`SGL Bit Bucket`是什么鬼？它只对Host读有用，用以告诉SSD，你往这个内存写入的东西我是不要的。好吧，你既然不要，我也就不传了。说了这么多，可能有点晕，结合下张图，可能会更明白点
![](/img/article/读NVMe之PRP与SGL/99b.png)

如果还是晕，看个例子吧。这个例子中，假设Host需要往SSD中读取13KB的数据，其中真正只需要11KB数据，这11KB的数据需要放到3个大小不同的内存中，分别是：3KB,4KB和4KB

![](/img/article/读NVMe之PRP与SGL/10b.png)

无论是PRP还是SGL，本质都是描述内存中的一段数据空间，这段数据空间在物理上可能连续的，也可能是不连续的。Host在命令中设置好PRP或者SGL，告诉SSD数据源在内存的什么位置，或者从闪存上读取的数据应该放到内存的什么位置。大家也许跟我有个同样的疑问(自作多情？)，那就是，既然有PRP，为什么还需要SGL?事实上，NVMe1.0的时候的确只有PRP，SGL是NVMe1.1之后引入的。SGL和PRP本质的区别在哪？下图道出了真相：一段数据空间，对PRP来说，它只能映射到一个个物理页，而对SGL来说，它可以映射到任意大小的连续物理空间

![](/img/article/读NVMe之PRP与SGL/13b.png)

**参考资料：**
[蛋蛋读NVMe之三](http://www.ssdfans.com/blog/2017/08/03/%E8%9B%8B%E8%9B%8B%E8%AF%BBnvme%E4%B9%8B%E4%B8%89/)

## "I want to say"
>本博文的学习了解到NVMe中如何描述内存数据存放的位置，并如何去获取？我们可以通过PRP指针及其链表与SGL的SGLSegment中的SGL Descriptor来表述
>
>本篇博文写的过程中很大一部分参考了上述参考资料，并结合自己阅读NVMe1.3c版本的标准，在阅读过程中学习到如何对博文书写得更加生动形象
>
>`快点看paper，快点写博文，quick！`
>`想想要不要立个Flag，But立flag这种事情还是谨慎为妙哈哈！不了不了…`