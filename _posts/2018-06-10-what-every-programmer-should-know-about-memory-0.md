---
layout: post
title: 每个程序员都应该了解的内存知识-0
categories: [linux, memory]
description: linux memory
keywords: linux memory
---

### 注
这个系列文章源于`What Every Programmer Should Know About Memory`[^1]，粗读下来觉得很不错，最好能留存下来。同时发现这个系列的文章已经有一部分被人翻译了。故在此转发留存一份，毕竟存在自己收留的才是最可靠的，经常发现很多不错的文章链接失效的情况。

本文转载自[^2]，翻译自[^3]。本人进行了轻微的修改，感觉更符合原义。

# 1 简介
早期计算机比现在更为简单。系统的各种组件例如CPU，内存，大容量存储器和网口，由于被共同开发因而有非常均衡的表现。例如，内存和网口并不比CPU在提供数据的时候更（特别的）快。

曾今计算机稳定的基本结构悄然改变，硬件开发人员开始致力于优化单个子系统。于是电脑一些组件的性能大大的落后因而成为了瓶颈。由于开销的原因，大容量存储器和内存子系统相对于其他组件来说改善得更为缓慢。

大容量存储的性能问题往往靠软件来改善: 操作系统将常用（且最有可能被用）的数据放在主存中，因为后者的速度要快上几个数量级。或者将缓存加入存储设备中，这样就可以在不修改操作系统的前提下提升性能。（然而，为了在使用缓存时保证数据的完整性，仍然要作出一些修改）。这些内容不在本文的谈论范围之内，就不作赘述了。

而解决内存的瓶颈更为困难，它与大容量存储不同，几乎每种方案都需要对硬件作出修改。目前，这些变更主要有以下这些方式:

* RAM的硬件设计(速度与并发度)
* 内存控制器的设计
* CPU缓存
* 设备的直接内存访问(DMA)

本文主要关心的是CPU缓存和内存控制器的设计。在讨论这些主题的过程中，我们还会研究DMA。不过，我们首先会从当今商用硬件的设计谈起。这有助于我们理解目前在使用内存子系统时可能遇到的问题和限制。我们还会详细介绍RAM的分类，说明为什么会存在这么多不同类型的内存。

本文不会包括所有内容，也不会包括最终性质的内容。我们的讨论范围仅止于商用硬件，而且只限于其中的一小部分。另外，本文中的许多论题，我们只会点到为止，以达到本文目标为标准。对于这些论题，大家可以阅读其它文档，获得更详细的说明。

当本文提到操作系统特定的细节和解决方案时，针对的都是Linux。无论何时都不会包含别的操作系统的任何信息，作者无意讨论其他操作系统的情况。如果读者认为他/她不得不使用别的操作系统，那么必须去要求供应商提供其操作系统类似于本文的文档。

在开始之前最后的一点说明，本文包含大量出现的术语“通常”和别的类似的限定词。这里讨论的技术在现实中存在于很多不同的实现，所以本文只阐述使用得最广泛最主流的版本。在阐述中很少有地方能用到绝对的限定词。

## 1.1 文档结构
这个文档主要视为软件开发者而写的。本文不会涉及太多硬件细节，所以喜欢硬件的读者也许不会觉得有用。但是在我们讨论一些有用的细节之前，我们先要描述足够多的背景。

在这个基础上，本文的第二部分将描述RAM（随机寄存器）。懂得这个部分的内容很好，但是此部分的内容并不是懂得其后内容必须部分。我们会在之后引用不少之前的部分，所以心急的读者可以跳过任何章节来读他们认为有用的部分。

第三部分会谈到不少关于CPU缓存行为模式的内容。我们会列出一些图标，这样你们不至于觉得太枯燥。第三部分对于理解整个文章非常重要。第四部分将简短的描述虚拟内存是怎么被实现的。这也是你们需要理解全文其他部分的背景知识之一。

第五部分会提到许多关于Non Uniform Memory Access (NUMA)系统。

第六部分是本文的中心部分。在这个部分里面，我们将回顾其他许多部分中的信息，并且我们将给阅读本文的程序员许多在各种情况下的编程建议。如果你真的很心急，那么你可以直接阅读第六部分，并且我们建议你在必要的时候回到之前的章节回顾一下必要的背景知识。

本文的第七部分将介绍一些能够帮助程序员更好的完成任务的工具。即便在彻底理解了某一项技术的情况下，距离彻底理解在非测试环境下的程序还是很遥远的。我们需要借助一些工具。

第八部分，我们将展望一些在未来我们可能认为好用的科技。

## 1.2 反馈问题
作者会不定期更新本文档。这些更新既包括伴随技术进步而来的更新也包含更改错误。非常欢迎有志于反馈问题的读者发送电子邮件。

## 1.3 致谢
我首先需要感谢Johnray Fuller尤其是Jonathan Corbet，感谢他们将作者的英语转化成为更为规范的形式。Markus Armbruster提供大量本文中对于问题和缩写有价值的建议。

## 1.4 关于本文
本文题目对David Goldberg的经典文献《What Every Computer Scientist Should Know About Floating-Point Arithmetic》[goldberg]表示致敬。Goldberg的论文虽然不普及，但是对于任何有志于严格编程的人都会是一个先决条件

# 2 商用硬件现状
鉴于目前专业硬件正在逐渐淡出，理解商用硬件的现状变得十分重要。现如今，人们更多的采用水平扩展，也就是说，用大量小型、互联的商用计算机代替巨大、超快(但超贵)的系统。原因在于，快速而廉价的网络硬件已经崛起。那些大型的专用系统仍然有一席之地，但已被商用硬件后来居上。2007年，Red Hat认为，未来构成数据中心的“积木”将会是拥有最多4个插槽的计算机，每个插槽插入一个四核CPU，这些CPU都是超线程的。（超线程使单个处理器核心能同时处理两个以上的任务，只需加入一点点额外硬件）。也就是说，这些数据中心中的标准系统拥有最多64个虚拟处理器（至今来看2018那年，96核/128核的服务已经是很常见的服务器配置了）。当然可以支持更大的系统，但人们认为4插槽、4核CPU是最佳配置，绝大多数的优化都针对这样的配置。

在不同商用计算机之间，也存在着巨大的差异。不过，我们关注在主要的差异上，可以涵盖到超过90%以上的硬件。需要注意的是，这些技术上的细节往往日新月异，变化极快，因此大家在阅读的时候也需要注意本文的写作时间。

这么多年来，个人计算机和小型服务器被标准化到了一个芯片组上，它由两部分组成: 北桥和南桥，见图2.1。

![图2.1 北桥和南桥组成的结构](/images/posts/memory/cpumemory.4.png)

图2.1 北桥和南桥组成的结构

CPU通过一条通用总线(前端总线，FSB)连接到北桥。北桥主要包括内存控制器和其它一些组件，内存控制器决定了RAM芯片的类型。不同的类型，包括DRAM、Rambus和SDRAM等等，要求不同的内存控制器。

为了连通其它系统设备，北桥需要与南桥通信。南桥又叫I/O桥，通过多条不同总线与设备们通信。目前，比较重要的总线有PCI、PCI Express、SATA和USB总线，除此以外，南桥还支持PATA、IEEE 1394、串行口和并行口等。比较老的系统上有连接北桥的AGP槽。那是由于南北桥间缺乏高速连接而采取的措施。现在的PCI-E都是直接连到南桥的。

这种结构有一些需要注意的地方:

* 从某个CPU到另一个CPU的数据需要走它与北桥通信的同一条总线。
* 与RAM的通信需要经过北桥
* RAM只有一个端口。(本文不会介绍多端口RAM，因为商用硬件不采用这种内存，至少程序员无法访问到。这种内存一般在路由器等专用硬件中采用。)
* CPU与南桥设备间的通信需要经过北桥

在上面这种设计中，瓶颈马上出现了。第一个瓶颈与设备对RAM的访问有关。早期，所有设备之间的通信都需要经过CPU，结果严重影响了整个系统的性能。为了解决这个问题，有些设备加入了直接内存访问(DMA)的能力。DMA允许设备在北桥的帮助下，无需CPU的干涉，直接读写RAM。到了今天，所有高性能的设备都可以使用DMA。虽然DMA大大降低了CPU的负担，却占用了北桥的带宽，与CPU形成了争用。

第二个瓶颈来自北桥与RAM间的总线。总线的具体情况与内存的类型有关。在早期的系统上，只有一条总线，因此不能实现并行访问。近期的RAM需要两条独立总线(或者说通道，DDR2就是这么叫的，见图2.8)，可以实现带宽加倍。北桥将内存访问交错地分配到两个通道上。更新的内存技术(如FB-DRAM)甚至加入了更多的通道。

由于带宽有限，我们需要以一种使延迟最小化的方式来对内存访问进行调度。我们将会看到，处理器的速度比内存要快得多，需要等待内存。如果有多个超线程核心或CPU同时访问内存，等待时间则会更长。对于DMA也是同样。

除了并发以外，访问模式也会极大地影响内存子系统、特别是多通道内存子系统的性能。关于访问模式，可参见2.2节。

在一些比较昂贵的系统上，北桥自己不含内存控制器，而是连接到外部的多个内存控制器上(在下例中，共有4个)。

![图2.2 拥有外部控制器的北桥](/images/posts/memory/cpumemory.5.png)

图2.2 拥有外部控制器的北桥

这种架构的好处在于，多条内存总线的存在，使得总带宽也随之增加了。而且也可以支持更多的内存。通过同时访问不同内存区，还可以降低延时。对于像图2.2中这种多处理器直连北桥的设计来说，尤其有效。而这种架构的局限在于北桥的内部带宽，非常巨大(来自Intel)。（出于完整性的考虑，还需要补充一下，这样的内存控制器布局还可以用于其它用途，比如说「内存RAID」，它可以与热插拔技术一起使用。）

使用外部内存控制器并不是唯一的办法，另一个最近比较流行的方法是将控制器集成到CPU内部，将内存直连到每个CPU。这种架构的走红归功于基于AMD Opteron处理器的SMP系统。图2.3展示了这种架构。Intel则会从Nehalem处理器开始支持通用系统接口(CSI)，基本上也是类似的思路——集成内存控制器，为每个处理器提供本地内存。

![图2.3 集成的内存控制器](/images/posts/memory/cpumemory.6.png)

图2.3 集成的内存控制器

通过采用这样的架构，系统里有几个处理器，就可以有几个内存库(memory bank)。比如，在4 CPU的计算机上，不需要一个拥有巨大带宽的复杂北桥，就可以实现4倍的内存带宽。另外，将内存控制器集成到CPU内部还有其它一些优点，这里就不赘述了。

同样也有缺点。首先，系统仍然要让所有内存能被所有处理器所访问，导致内存不再是统一的资源(NUMA即得名于此)。处理器能以正常的速度访问本地内存(连接到该处理器的内存)。但它访问其它处理器的内存时，却需要使用处理器之间的互联通道。比如说，CPU<sub>1</sub>如果要访问CPU<sub>2</sub>的内存，则需要使用它们之间的互联通道。如果它需要访问CPU<sub>4</sub>的内存，那么需要跨越两条互联通道。

使用互联通道是有代价的。在讨论访问远端内存的代价时，我们用「NUMA因子」这个词。在图2.3中，每个CPU有两个层级: 相邻的CPU，以及两个互联通道外的CPU。在更加复杂的系统中，层级也更多。甚至有些机器有不止一种连接，比如说IBM的x445和SGI的Altix系列。CPU被归入节点，节点内的内存访问时间是一致的，或者只有很小的NUMA因子。而在节点之间的连接代价很大，而且有巨大的NUMA因子。

目前，已经有商用的NUMA计算机，而且它们在未来应该会扮演更加重要的角色。人们预计，从2008年底开始，每台SMP机器都会使用NUMA。每个在NUMA上运行的程序都应该认识到NUMA的代价。在第5节中，我们将讨论更多的架构，以及Linux内核为这些程序提供的一些技术。

除了本节中所介绍的技术之外，还有其它一些影响RAM性能的因素。它们无法被软件所左右，所以没有放在这里。如果大家有兴趣，可以在第2.1节中看一下。介绍这些技术，仅仅是因为它们能让我们绘制的RAM技术全图更为完整，或者是可能在大家购买计算机时能够提供一些帮助。

以下的两节主要介绍一些入门级的硬件知识，同时讨论内存控制器与DRAM芯片间的访问协议。这些知识解释了内存访问的原理，程序员可能会得到一些启发。不过，这部分并不是必读的，心急的读者可以直接跳到第2.2.5节。

## 2.1 RAM类型
这些年来，出现了许多不同类型的RAM，各有差异，有些甚至有非常巨大的不同。那些很古老的类型已经乏人问津，我们就不仔细研究了。我们主要专注于几类现代RAM，剖开它们的表面，研究一下内核和应用开发人员们可以看到的一些细节。

第一个有趣的细节是，为什么在同一台机器中有不同的RAM？或者说得更详细一点，为什么既有静态RAM(SRAM {SRAM还可以表示「同步内存」。})，又有动态RAM(DRAM)。功能相同，前者更快。那么，为什么不全部使用SRAM？答案是，代价。无论在生产还是在使用上，SRAM都比DRAM要贵得多。生产和使用，这两个代价因子都很重要，后者则是越来越重要。为了理解这一点，我们分别看一下SRAM和DRAM一个位的存储的实现过程。

在本节的余下部分，我们将讨论RAM实现的底层细节。我们将尽量控制细节的层面，比如，在「逻辑的层面」讨论信号，而不是硬件设计师那种层面，因为那毫无必要。

### 2.1.1 静态RAM

![图2.4 6-T静态RAM](/images/posts/memory/cpumemory.7.png)

图2.4 6-T静态RAM

图2.4展示了6晶体管SRAM的一个单元。核心是4个晶体管M<sub>1</sub> - M<sub>4</sub>，它们组成两个交叉耦合的反相器。它们有两个稳定的状态，分别代表0和1。只要保持V<sub>dd</sub>有电，状态就是稳定的。

当访问单元的状态时，需要拉升WL的电平。使得$${BL}$$和$$\overline{BL}$$上可以读取状态。如果需要覆盖单元状态，先将$${BL}$$和$$\overline{BL}$$设置为期望的值，然后升起WL电平。由于外部的驱动强于内部的4个晶体管（M<sub>1</sub> - M<sub>4</sub>），所以旧状态会被覆盖。

更多详情，可以参考[sramwiki]。为了下文的讨论，需要注意以下问题:

* 一个单元需要6个晶体管。也有采用4个晶体管的SRAM，但有缺陷。
* 维持状态需要恒定的电源。
* 升起WL后立即可以读取状态。信号与其它晶体管控制的信号一样，是直角的(快速在两个状态间变化)。
* 状态稳定，不需要刷新循环。

SRAM也有其它形式，不那么费电，但比较慢。由于我们需要的是快速RAM，因此不在关注范围内。这些较慢的SRAM的主要优点在于接口简单，比动态RAM更容易使用。

### 2.1.2 动态RAM
动态RAM比静态RAM要简单得多。图2.5展示了一种普通DRAM的结构。它只含有一个晶体管和一个电容器。显然，这种复杂性上的巨大差异意味着功能上的迥异。

![图2.5 1-T动态RAM](/images/posts/memory/cpumemory.8.png)

图2.5 1-T动态RAM

动态RAM的状态是保持在电容器C中。晶体管M用来控制访问。如果要读取状态，拉升访问线AL，这时，可能会有电流流到数据线DL上，也可能没有，取决于电容器是否有电。如果要写入状态，先设置DL，然后升起AL一段时间，直到电容器充电或放电完毕。

动态RAM的设计有几个复杂的地方。由于读取状态时需要对电容器放电，所以这一过程不能无限重复，不得不在某个点上对它重新充电。

更糟糕的是，为了容纳大量单元(现在一般在单个芯片上容纳10的9次方以上的RAM单元)，电容器的容量必须很小(0.000000000000001法拉以下)。这样，完整充电后大约持有几万个电子。即使电容器的电阻很大(若干兆欧姆)，仍然只需很短的时间就会耗光电荷，称为「泄漏」。

这种泄露就是现在的大部分DRAM芯片每隔64ms就必须进行一次刷新的原因。在刷新期间，对于该芯片的访问是不可能的，这甚至会造成半数任务的延宕。（相关内容请察看【highperfdram】一章）

这个问题的另一个后果就是无法直接读取芯片单元中的信息，而必须通过信号放大器将0和1两种信号间的电势差增大。

最后一个问题在于电容器的冲放电是需要时间的，这就导致了信号放大器读取的信号并不是典型的矩形信号。所以当放大器输出信号的时候就需要一个小小的延宕，相关公式如下

![](/images/posts/memory/capcharge.png)

这就意味着需要一些时间（时间长短取决于电容C和电阻R）来对电容进行冲放电。另一个负面作用是，信号放大器的输出电流不能立即就作为信号载体使用。图2.6显示了冲放电的曲线，x轴表示的是单位时间下的R*C。

![](/images/posts/memory/cpumemory.57.png)

与静态RAM可以即刻读取数据不同的是，当要读取动态RAM的时候，必须花一点时间来等待电容的冲放电完全。这一点点的时间最终限制了DRAM的速度。

当然了，这种读取方式也是有好处的。最大的好处在于缩小了规模。一个动态RAM的尺寸是小于静态RAM的。这种规模的减小不单单建立在动态RAM的简单结构之上，也是由于减少了静态RAM的各个单元独立的供电部分。以上也同时导致了动态RAM模具的简单化。

综上所述，由于不可思议的成本差异，除了一些特殊的硬件（包括路由器什么的）之外，我们的硬件大多是使用DRAM的。这一点深深的影响了咱们这些程序员，后文将会对此进行讨论。在此之前，我们还是先了解下DRAM的更多细节。

### 2.1.3 DRAM 访问
一个程序选择了一个内存位置使用到了一个虚拟地址。处理器转换这个到物理地址最后将内存控制选择RAM芯片匹配了那个地址。在RAM芯片去选择单个内存单元，部分的物理地址以许多地址行的形式被传递。

它单独地去处理来自于内存控制器的内存位置将完全不切实际：4G的RAM将需要 $2^32$ 地址行。地址传递DRAM芯片的这种方式首先必须被路由器解析。一个路由器的N多地址行将有$2^N$输出行。这些输出行能被使用到选择内存单元。使用这个直接方法对于小容量芯片不再是个大问题。



# 参考
[^1]: [What-Every-Programmer-Should-Know-About-Memory](/images/posts/memory/What-Every-Programmer-Should-Know-About-Memory.pdf)
[^2]: [每个程序员都应该了解的内存知识【第一部分】](https://www.oschina.net/translate/what-every-programmer-should-know-about-memory-part1)
[^3]: [What every programmer should know about memory, Part 1](http://lwn.net/Articles/250967/)