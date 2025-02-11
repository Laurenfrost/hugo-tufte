---
title: "Towards Transparent CPU Scheduling"
date: 2020-08-18 17:31:35
meta: true
math: true
subtitle: "透明的 CPU 调度"
toc: true
categories: 
    - 操作系统
    - 论文阅读
---

最近在看 [*OSTEP*](http://pages.cs.wisc.edu/~remzi/OSTEP/)，作者在进程调度部分推荐了该论文：

[Joseph T. Meehean, *Towards Transparent CPU Scheduling*, University of Wisconsin–Madison, 2011](http://www.cs.wisc.edu/adsl/Publications/meehean-thesis11.pdf)  

于是我在有道翻译的帮助下翻译该博士论文，有些地方本人进行了润色。

---

本文提出用科学的方法对 CPU 调度器有更深入的了解； 我们使用这种方法来解释和理解 CPU 调度器有时不稳定的行为。这种方法首先将受控的工作负载引入商品操作系统，并观察 CPU 调度器的行为。__通过这些观察__，我们能够推断底层 CPU 调度策略，并创建预测调度行为的模型。

在将科学分析应用于 CPU 调度器方面，我们已经取得了两项进展。首先是 CPU Futures，这是一个嵌入到 CPU 调度器和用户空间控制器中的预测调度模型的组合，后者使用来自这些模型的反馈来指导应用程序。我们基于两种不同的调度范例(分时和比例-份额)，为两种不同的 Linux 调度器(CFS 和 O(1))开发了这些预测模型。通过三个不同的案例研究，我们演示了应用程序可以使用我们的预测模型将来自低重要性应用程序的干扰减少 70% 以上，将 Web 服务器的供应不足减少一个数量级，并执行与 CPU 调度器相矛盾的调度策略。

我们的第二个贡献是 *Harmony*。这是一个框架和一组实验，用于从普通操作系统中提取多处理器调度策略。我们使用这个工具提取和分析三个 Linux 调度器的策略：O(1)、CFS 和 BFS。这些调度器通常执行截然不同的策略。从高层次的角度来看，O(1) 调度器谨慎地选择要迁移的进程,更看重{{<ruby mark="Processor Affinity">}}处理器亲缘性{{</ruby>}}。相反，CFS 则不断地寻找更好的平衡，最终选择了随机迁移进程。而 BFS 则非常看重公平性，经常忽略处理器关联性。

---

## 导论

{{< epigraph pre="Thucydides" >}}
把自己渴望的东西托付给漫不经心的希望，用至高无上的理由把自己不渴望的东西推到一边。这就是人类的习性。
{{< /epigraph >}}

目前，应用程序、开发人员和系统研究人员都将最佳工作 CPU 调度器视为可靠的黑箱。依赖这些调度器来提供底层但至关重要的服务。在过去的几十年里，处理器快速增长的性能支持了这种依赖和映像。

单处理器速度不断增长的结束意味着这些假设应该重新考虑。如果对中等负载下的CPU调度器进行更仔细的检查，就会发现它们可以提供反复无常的服务，并实现与依赖于它们的应用程序冲突的资源争用策略。这种看似不稳定的行为对应用程序的可感知可靠性有可怕的后果；它会导致缓慢的响应时间，病态的行为，甚至应用程序失败。

在这项工作中，我们建议采取科学的方法来理解CPU调度程序；我们开始通过收集观察、生成假设和生成预测模型来揭开CPU调度器的神秘面纱。例如，对 CPU 调度的进一步研究表明，它有时不稳定的行为并不是不可预测的，而只是几个善意策略的不幸组合结果。这种方法产生的观察结果和预测模型使 CPU 调度对应用程序更加透明(并且可由应用程序控制)，并使研究者们能够改进或细化调度策略。

本文讨论了我们在科学分析 CPU 调度方面所做的两个改进。在第一个 CPU Futures 中，我们创建并演示嵌入到 CPU 调度器中的预测模型的价值。应用程序使用这些预测模型来主动引导两个当代的 Linux CPU 调度程序实现它们所期望的调度策略。在第二部分，我们实现了一个多处理器 CPU 调度观察的收集框架。然后，我们使用这个名为 *Harmony* 的框架从三个 Linux CPU 调度器(O(1)、CFS 和 BFS)提取 CPU 调度策略。

本章对驱动这些理论的思想进行了高层次的概述。第一部分讨论 CPU 调度日益增加的重要性。下一节将介绍黑箱 CPU 调度造成的困难。在本节之后，将使用科学的方法进行更广泛的讨论，以提高 CPU 调度器的透明度。最后，我们对本文的结构进行了概述。

### CPU 调度的重要性

在过去的 15 年里， CPU 调度算法的进步已经很大程度上与处理器快速增长的速度无关(有些例外[<sup>27</sup>](#refer-27) [<sup>48</sup>](#refer-48) [<sup>56</sup>](#refer-56) [<sup>65</sup>](#refer-65))。在多媒体应用程序的实时调度领域中，许多有趣的工作只是被新处理器的速度超越了[<sup>24</sup>](#refer-24) [<sup>25</sup>](#refer-25) [<sup>26</sup>](#refer-26) [<sup>28</sup>](#refer-28)。每一台现代的台式电脑都可以轻松地播放流媒体，而无需专门为此目的而设计的研究技术的帮助。事实上，许多电脑依然在使用 80 年代和 90 年代早期开发的分时调度技术来播放实时媒体。那么，为什么系统研究团体应该投入时间和金钱来研究任务调度呢?

答案是免费的硬件之旅已经结束，单核处理器速度已在 2003 年达到顶峰[<sup>63</sup>](#refer-63)。当前硬件的趋势是更多的处理器核数，而不是更快的单核处理器，这意味着CPU调度再次重要了起来。为了改进多核处理器下应用程序的性能，需要改进CPU调度算法。

多核处理器不会像更快的单核处理器那样自动地为应用程序提供性能改进。相反，必须重新设计应用程序以增加其并行性。同意，CPU 调度器也必须要重新设计，以最大限度地提高这种新的应用程序并行性的性能。CPU 调度策略(在很大程度上是一种机制)对于在自己的机器上运行的串行应用程序并不重要。然而，现在，一个应用程序可能在一台机器上与自身的多个并发实例竞争/合作。在这个场景中，CPU 调度就变得非常重要。如果应用程序的某些部分没有响应，而另一些部分运行良好，那么应用程序可能会出现无响应、不稳定甚至造成严重故障。

如果 CPU 未被充分利用，CPU 调度几乎无关紧要。事实上，在未充分利用的 CPU 上，调度器的存在只会影响调度延迟。人们最初的期望是，增加的每台机器的处理器数量可以减少潜在的CPU争用，从而减少对良好 CPU 调度的需求。然而，冗余硬件的增加是与{{<ruby mark="Server Consolidation">}}服务器整合{{</ruby>}}的增加同时发生的，这重新引入了 CPU 争用的可能性。

为了增加利润并提高(集群的)吞吐量，服务器整合可能会减少应用程序的硬件分配，直到应用程序以接近其分配的硬件容量运行为止。这种对资源分配的精确调整将部分调度问题转移到了应用程序的操作系统上。在接近系统满负荷的情况下运行意味着即使负载稍有增加，也可能使系统处于过载状态。而一旦系统过载， CPU 调度就变得至关重要。因为此时 CPU 调度器必须仔细决定如何分配其有限的资源。

### 不透明的 CPU 调度程序

普通 CPU 调度器非常不透明。现在正在运行的大部分系统中，来自 CPU 调度器的主要反馈只是其策略的直接结果：每个应用程序分配了多少 CPU。虽然总比没有好，但这种反馈传达的信息非常少。它没有提供关于为什么给应用程序一个特定分配的信息。这就是应用程序想要的吗？是否存在 CPU 争用？由于 CPU 争用，应用程序的运行速度会慢多少？如果应用程序有更高或更低的优先级，它会收到什么？

这种不透明性对用户、应用程序、应用程序开发人员和系统研究人员产生了负面影响。用户和应用程序无法确定他们的程序运行将怎样被 CPU 调度和争用所影响。对于运行缓慢的应用程序，用户甚至可能根本无法判断这是 CPU 本身的锅还是 CPU 调度器的锅。应用程序无法确定系统是否能够支持其当前的并行度级别，应用程序无法确定是否应该增加或减少并行度，更不要说确定增加或减少多少。这些问题常常使 CPU 调度器显得不可预测和不一致。

对 CPU 调度器的离线分析证明，它的不透明性既有所增加，亦有所减少。一般操作系统的单处理器调度策略通常有很好的文档，主要在教科书[<sup>20</sup>](#refer-20) [<sup>35</sup>](#refer-35) [<sup>88</sup>](#refer-88) [<sup>89</sup>](#refer-89) [<sup>108</sup>](#refer-108) [<sup>111</sup>](#refer-111)中。然而，这些操作系统的多处理器调度策略常常缺少文档记录或者根本没有文档记录；发行主发行版 Linux 调度器时，没有任何关于其多处理器调度策略的文档。此外，常用的单处理器和多处理器调度策略的文档常常以实现为重点。这让读者大致了解了如何构建调度程序，但无法了解在给定特定工作负载时它的行为。

应用程序开发人员无法在其应用程序中有效地构建并行性，因为他们无法预测 CPU 调度器将如何管理这种并行性。相反，他们必须使用试验和错误来设计他们的应用程序。这种不确定性在实现结束时增加了一个额外的递归调试步骤，而不是允许应用程序设计人员基于很容易理解的 CPU 调度器参数构建应用程序。

最后，如果系统研究人员对当前技术没有一个坚实的了解，那么调度改进是困难的。有限的商品调度文件意味着每个研究人员必须从头开始发展这一理解。此外，如果没有有用的运行时反馈，用户和应用程序开发人员就无法提供关于他们使用普通 CPU 调度器时遇到的问题的详细说明。这些限制使得系统研究人员很难更改 CPU 调度器以更接近应用程序和用户的需求。

CPU 调度器的不透明性质让用户、开发人员和系统研究人员只能猜测系统失败的原因以及如何改进它们。这种“粗心的希望”是好的科学和工程的对立面。

### 增加 CPU 调度的透明度

要解决不透明调度器问题，不能简单地公开它们的内部工作，而是要形成对它们行为的深入理解，进而产生更有价值的信息。例如，“给用户空间展示 CPU 调度器所调度的运行队列”，这种就没有什么用，因为这种轻微的透明度（*原文： micro-transparency*）只能传递“接下来会运行哪个进程”这种没有什么用的信息。至于“各个进程能获得多少空间分配”和“队列里的最后一个进程要排多久的队”，进程则一无所知。要是进程修改了它自己的优先级或行为，进程就更不可能知道这些指标将如何变化。

我们认为，CPU 调度程序应该像研究自然系统一样：运用科学方法。第一步是观察给定各种工作负载(刺激)时调度器的行为。接下来，我们对观察到的行为做出假设。然后，使用这些假设作为基础，我们生成预测模型。最后，我们将预测模型的结果与所观察的系统进行比较，并完善我们的假设和模型。在此分析中创建的假设和模型不仅使我们能够预测CPU调度器的行为，还能使我们对 CPU 调度策略的行为有了基本的理解。这种理解和预测能力将把 CPU 调度器从难以理解的黑盒子转变为可理解的透明系统。

重要的是，我们不仅要了解 CPU 调度器实现的策略，还要了解该策略对实际工作负载的影响。策略通常是根据设计师关于如何满足高级目标的直觉创建的。为了理解调度策略，在某些情况下，我们必须从观察到的 CPU 调度程序的行为中反向工程高级目标。然后我们可以分析该策略满足目标的程度。分析这项政策可能产生的任何副作用也是至关重要的。这也是基于对 CPU 调度器行为的观察。一旦我们有了更深层次的理解，我们就可以开始提前预测给定特定工作负载的调度器的行为。

由这些观察和预测模型所产生的增加的透明性为系统研究人员和应用程序开发人员提供了提高性能和可靠性的能力。系统研究人员可以使用观察和假设来提出对调度政策的改进。应用程序开发人员可以使用预测模型来指导其应用程序的开发；开发人员可以将他们的应用程序转向 CPU 调度器中工作最好的部分，同时尝试减轻调度器的缺点。

在运行系统中嵌入调度模型可以为用户和应用程序提供非常需要的反馈。用户可以确切地知道由于 CPU 争用或选择的 CPU 调度器不佳而导致的性能下降。系统管理员可以将此信息用于性能调试或资源规划。应用程序也可以使用此反馈来密切监视其并发工作流的性能，并确保每个工作流都取得了足够的进展。然后，应用程序可以使用有关每个工作流的相对重要性的特定于应用程序的逻辑来解决由 CPU 争用引起的问题。

我们对 CPU 调度的科学分析做出了两个贡献：CPU Futures 和 Harmony。CPU Futures 是嵌入到 CPU 调度器中的一组预测模型和一个用户空间控制器的组合，用于使用来自这些模型的反馈来引导应用程序。我们已经为两种一般类型的 CPU 调度器(分时和比例共享)创建了这些预测调度模型，并为两种 Linux 调度器(CFS 和 O(1))实现了它们。将这些模型与一个简单的用户空间控制器结合起来，我们将展示它们对于分布式应用程序和低重要性的后台应用程序的价值。

Harmony 是一个为 CPU 调度器生成刺激(合成工作负载)并观察结果行为的实验框架。这个框架只需要少量的低级插装，并且不依赖于操作系统文档或源代码。我们还设计了一组实验来从商用 CPU 调度器中提取多处理器调度策略。通过这些实验，我们演示了 Harmony 的价值，并开始说明三个 Linux 调度器的调度策略：O(1)、CFS 和 BFS。

### 本文结构

在接下来的两章中，我们将提供 CPU 调度的背景资料。在第 2 章中，我们讨论了 CPU 调度器和多处理器调度架构的流行类型。第 3 章提供了对不透明 CPU 调度器问题的更深入的分析，包括激励实验。

再往后的两章介绍了 CPU Futures，这是一组 CPU 调度器的预测模型，它允许应用程序执行它们自己的调度策略。第 4 章概述了 CPU Futures，并详细讨论了调度模型。在第 5 章中，我们用三个案例来说明如何使用 CPU Futures 反馈，并说明嵌入式调度模型的有用性。

Harmony 将在那之后的两章中介绍。第 6 章概述了 Harmony ，以及使用 Harmony 从三个 Linux 调度器(O(1)、CFS 和 BFS)提取基本策略的结果。第 7 章则从这三个调度程序中引出更复杂的策略。

在最后两章，我们讨论了本论文的背景和贡献。第 8 章介绍与 Harmony 及 CPU Futures 有关的工作。在第 9 章中，我们讨论了我们的结论并总结了这项工作的贡献。

## CPU 调度

> Some have two feet  
> And some have four.  
> Some have six feet  
> And some have more.  
> — Dr. Seuss (One Fish, Two Fish, Red Fish, Blue Fish)  

CPU 调度器的目标是提供一种假象，即每个进程或线程都有自己的专用 CPU。虚拟化 CPU 所需的机制相当简单。创建一个策略来在竞争的进程和线程之间划分物理 CPU 是一个困难得多的问题。详细地理解这个问题对于理解提高 CPU 调度器透明度的重要性至关重要，无论是通过改进的接口还是经验观察。

CPU 调度没有计划；没有最佳的解决方案。相反，CPU 调度是关于平衡目标和做出困难的权衡。确定 CPU 调度器的底层目标是了解其复杂行为的关键。了解调度器为实现其目标所做的权衡对于理解商用调度器的发展是至关重要的。新的调度器要么强调不同的目标，要么提供一种更简单的方法来实现与其前身相同的目标。理解商品调度程序是改进它们的第一步。

人们应当努力避免在调度策略搞主观价值判断。不付出任何代价、不强调一个目标且不改进另一个目标的调度策略是最糟糕的。例如，人们可能会因为太复杂而忽略现成的商业调度器。Butler Lampson 曾经提倡回归简单的三层循环调度器[<sup>85</sup>](#refer-85)。Lampson 的调度器不会比商业调度器更好，它只是更加强调简单。

系统设计者创建 CPU 调度策略来是为了匹配特定的环境。调度策略中的每个折衷都反映了对工作负载和硬件配置的假设。因此，只能根据 CPU 调度器在给定环境中的工作情况来评估它。需要注意的是，<em><strong>通用</strong></em> 是一种环境选择；它只是封装了所有其他选择。（<i>原文：It is important to note that general-purpose is an environment choice; it merely encapsulates all other choices.</i>）

本章我们从描述组成典型分布式计算环境的硬件、操作系统和应用程序开始。然后我们对CPU调度程序进行分类，讨论每个类别如何实现它们的主要目标以及代价是什么。接下来，我们将解释多处理器系统如何引入一组新的冲突调度目标。在本节中，我们还将介绍两类多处理器调度器。最后，我们概述了普通操作系统中的 CPU 调度策略，重点介绍了 Linux 。

### 环境

任何脱离调度环境的 CPU 调度策略评估都是耍流氓；如果不了解目标硬件配置和软件栈，就不可能了解给定CPU调度策略的值。

现代计算严重依赖于分布式系统，而分布式系统又依赖于大型服务器类机器和集群来为成千上万的用户提供计算能力。这些机器往往有多个微处理器，每个微处理器包含几个同构处理核心。这些机器的内存结构通常是非统一内存访问(NUMA)。这些 NUMA 架构比之前几十年的 big iron 架构更紧密耦合(速度更快)；内存节点位于一个单一的主板和远程节点访问发生通过一个特殊的处理器到处理器总线[<sup>23</sup>](#refer-23) [<sup>46</sup>](#refer-46) [<sup>50</sup>](#refer-50) [<sup>133</sup>](#refer-133)。

这些服务器和集群运行各种操作系统，包括 Windows、Linux、FreeBSD 和 Solaris。本文主要关注 Linux，原因有三个。首先，它很流行：超过 41% 的 Web 服务器[<sup>96</sup>](#refer-96)和 91% 的世界 500 强计算机系统[<sup>117</sup>](#refer-117)运行 Linux。分析来自大型强子对撞机(迄今最昂贵的科学仪器之一)数据的网格计算软件仅在 Linux 上运行。其次，Linux 社区正在积极地考虑和开发 CPU 调度器。在 2010 年，一个 Linux 调度器有超过 223 个补丁，平均每 27 小时有一个补丁。这表明社区对改进 Linux CPU 调度非常感兴趣。最后，Linux 是开源的，这使得它更容易原型化和发布调度研究。

分布式系统通常有三个要求：高可靠、高并发、多用户。（<i>原文：Distributed systems are composed of multiple multifaceted distributed services: long-lived entities, often servicing multiple requests and users concurrently.</i>）这使得系统能够同时处理多个用户请求并执行后台处理。有三种架构能支持这个复杂的并发处理系统。需要注意的是，每种架构允许可变的并发量，可以为每个工作单元分配可变的重要级别，并且依赖于长时间运行的进程或线程。

第一种架构，使用多个进程管理并发性。例如，每当用户登录到邮件服务器时，Dovecot IMAP 服务器就会 fork 一个新的工作进程；登录后，这个工作进程处理所有用户的邮件请求[<sup>6</sup>](#refer-6)。

第二种架构，使用线程处理多个并发请求。Tomcat 应用服务器是用于创建 Web 服务的应用程序框架，它使用基于线程的体系结构。当 Web 请求到达时，就会从预先创建的池中选择一个线程为其[<sup>8</sup>](#refer-8)提供服务。

最后一种架构通常称为事件驱动，它基于使用单个、非阻塞进程串行地处理多个请求。例如，Tornado Web 服务器使用单个进程来处理传入的 Web 请求；这个单进程会在多个并发请求之间切换，或者在某个定义良好的边界上切换，或者在 CPU[<sup>9</sup>](#refer-9) 以外的资源上阻塞请求。

有些应用程序是混合使用这些体系结构设计的。例如，Condor 批处理系统的作业调度器主要实现为事件驱动的服务，但它也会 fork 进程来处理一些可能阻塞或需要太多作业调度器时间的任务[<sup>90</sup>](#refer-90)。

这些服务提供的大量并发性和各种各样的请求类型创建了一个环境，在这里，资源需求可能会意外地快速增长；服务也可能因用户数量的意外增加而超载。例如，一个流量相对较低的网页可能会在被一个新闻聚合网页（如 Slashdot）链接后突然成为互联网热点。

### CPU 调度

CPU 调度程序为应用程序提供了多个虚拟 CPU 的假象：每个应用程序似乎都有自己的 CPU 。因此，CPU 调度器的主要工作是在竞争任务{{<marginnote "原文注">}}原文注：这里的任务，在本文中作为一个通用术语，意思是进程或线程。{{</marginnote>}}之间安全地、最优地分配 CPU 资源。安全是由内核的上下文切换机制和内核代码划分成允许或不允许上下文切换的部分来实现的。优化更为困难，因为分配 CPU 资源的最佳方式可能因应用程序而异。因此，每个 CPU 调度器都需要一个特定于系统的策略来定义如何共享处理器。该策略包含了系统的广泛调度目标，并反映了系统对其工作负载的期望。

调度策略需要在相互冲突的目标之间进行平衡。现代调度策略主要权衡三个目标：公平性、调度延迟和进程执行速度。当然也存在其他目标，但这三个通常是最重要的。公平性是指 CPU 周期在某个时间尺度(例如一秒、一分钟、一小时)上的划分。任务在其时间段内的所获得的时钟周期称为该任务的 CPU 分配。公平没有定量的定义；事实上，每一种策略都定义了自己的 <em><strong>公平</strong></em> 分配模型。一项政策的公平性可以通过它与预期分配的匹配程度和时间尺度来衡量；时间尺度越小，公平感越强。

调度延迟是指任务在获得对 CPU 的控制权之前必须等待的时间。延迟对于交互任务来说是最重要的，因为高延迟会导致用户受挫。进度度量一个任务在给定时间内能够完成的工作。在称为饥饿（starvation）的极端情况下，任务可能根本没有进展。

调度策略必须在这些目标之间进行权衡。例如，为交互式任务划分优先级以减少延迟的调度策略可能会提供不公平的分配，甚至使得进程饥饿。另一个例子是，在小时间范围内提供公平分配的调度器可能会增加上下文切换的数量，从而影响进程执行速度。

CPU 调度器分为两大类：实时（real-time）和尽力而为（best-effort）。实时调度器保证了事件的响应时间；这些调度器确保系统始终满足应用程序定义的最大时延。实时调度器通常出现在需要延迟保证的环境中，比如机器人和嵌入式系统。为了提供这些保证，实时调度器需要知道所有应用程序的 CPU 分配和延迟需求。如果调度器不能提供应用程序所需的低延迟保证，则应用程序不会运行。这种控制策略限制了实时系统的并发性。

可以使用两种方法收集应用程序 CPU 延迟和分配需求。在第一种方法中，用户必须在启动应用程序之前指定这些值。对于不同的应用程序、硬件配置和输入，这些 CPU 需求是不同的。因此，即使对于每秒传输固定帧数的媒体播放器来说，确定这些需求也很困难。[<sup>28</sup>](#refer-28)

而其他实时调度器通过自动检测分配和延迟的需求来消除用户的负担[<sup>25</sup>](#refer-25) [<sup>26</sup>](#refer-26) [<sup>52</sup>](#refer-52) [<sup>130</sup>](#refer-130)。目前还不清楚这种替代技术是否适用于所有[<sup>28</sup>](#refer-28)的情况，到目前为止，它还没有出现在普通操作系统中。

与实时调度器相反，尽力而为调度器不提供任何保证。他们的主要目标是易于使用。因为它们只提供尽力而为的服务，所以不需要预先了解应用程序延迟或分配需求。尽力而为调度器也没有许可控制机制（*原文：admission control mechanisms*）来防止 CPU 争用。这些调度器可以在所有商用操作系统中找到，桌面和服务器类机器都使用它们。因为这是我们的目标环境，所以余下的工作将集中在尽力而为调度器上。

尽力而为调度器又可以分为三类：分时（Timesharing）、比例份额（Proportional-Share）和批处理（Batch）。以下各节将详细讨论分时共享和比例共享。批处理调度在我们的目标环境中并不常见，为了方便讨论，我们将其忽略。

#### 分时调度器

分时调度器的主要目标是为交互任务提供低延迟。这是通过根据交互级别将任务自动划分为类来实现的。一个任务的交互性越强，它就越快被安排。在这些调度器中，CPU 使用率是衡量交互性的主要指标；一个任务对 CPU 的要求越高，它的交互性就越低。注意，CPU 使用率和交互性之间的相关性是分时调度器做出的假设，而这种假设通常不适用于现代应用程序(例如电子游戏)。

公平性对于分时调度器也很重要。这类调度器没有一个公平模型，而是有一个公平思想。这种思想的核心是使用最少 CPU 的任务应该获得最多的 CPU 时间。作为这一核心信念的补充，分时节约思想通常允许用户以用户指定优先级的形式输入。具有最佳优先级的任务被分配尽可能多的 CPU。次优优先级从剩余优先级中分配，依此类推。当然，这些用户分配的优先级的主要目标是减少交互任务的延迟。分时调度器通常将它们作为参考。例如，如果一个 CPU 密集型任务被用户分配了比交互式任务更高的优先级，调度器仍然可以为交互式任务提供很好的服务。

分时调度器对进程执行速度并不关心，它只需保证不让任何任务饿死。跟其他的调度器思路一样，所谓的“不饿死”是一个模糊的概念，并没有绝对的定义。所以每个调度器都可以自由地解释什么算“饥饿”、以及如何防止它。例如，Solaris 的分时调度器试图确保每个任务每秒运行一次，以防止资源耗尽[<sup>88</sup>](#refer-88)。Linux 的分时调度器可能会忽略超过一分钟的任务[<sup>35</sup>](#refer-35)。鉴于“饥饿”的定义非常模糊，而且跟另外两个目标相比，进程执行速度并没有那么要紧，所以在实践中，饥饿预防通常由一个临时机制提供。该机制与核心调度机制一起工作，但游离在核心调度机制之外。

分时调度器通常用多个运行队列来实现。每个队列都代表一个优先级；调度器分配的优先级是通过结合用户分配的优先级、任务交互性和公平性目标来计算的。由于任务可以在队列之间移动，因此这种体系结构称为多级反馈队列。

任务可以以两种不同的方式在队列之间移动。经典方法中，每个任务都会根据它的优先级被分配到相应队列，并给它相应的 CPU 时间调度量。任务的优先级越高，给任务分配的调度量越小。如果任务在没有释放 CPU 的情况下耗光了它在本轮的 CPU 时间调度量（这说明这个任务可能是个计算密集程序），那么它将被转移到较低的优先级并分配一个较大的 CPU 时间调度量。通过这种方式，每个任务根据其 CPU 行为和用户分配的优先级被分配一个调度优先级。[<sup>20</sup>](#refer-20)

另一方面，多级反馈队列使用衰减方法监视时间超过一个调度量的任务。根据任务的长期 CPU 消耗为其分配合适的调度优先级。这种 CPU 消耗会周期性地衰减，以防止某个任务因为一次大规模的 CPU 活动爆发而无限期地受到惩罚。实际上，任务每次在 CPU 上的运行时间都会被削减。然后，在每个衰减周期(由调度程序定义)中，调度程序将每个任务的 CPU 总消耗除以一个策略定义的数字（大于 1）。调度程序入队一个任务，它将根据该任务的 CPU 总消耗分配优先级；消耗 CPU 越少的任务优先级越高。[<sup>60</sup>](#refer-60) [<sup>71</sup>](#refer-71) [<sup>76</sup>](#refer-76)

#### 比例份额

比例共享调度器的基本目标是提供公平的分配。比例共享调度器中的公平性由广义处理器共享(GPS)模型定义[101]。直观地说，GPS模型试图提供一种假象，即每个任务都有自己的CPU。这些每个任务的虚拟cpu运行速度较慢，这与系统中任务的数量成正比。例如，在一个有三个任务和一个3GHz处理器的系统中，每个任务都会像拥有自己的1GHz处理器一样进行处理。当然，这种模型在现实世界中是不可能实现的，在现实世界中，处理器一次只能分配给一个任务，非常小的调度量程会导致较差的缓存性能。因此，该模型被解释为定义分配给任务的相对CPU分配。如果所有任务都是相同的，那么在给定的时间段内，所有任务都应该获得相同的CPU分配。


## 参考文献

<div id="refer-1"></div>
<p>[1] Red hat enterprise linux life cycle. URL https://access.redhat.com/support/policy/updates/errata/.</p>

<div id="refer-2"></div>
<p>[2] From a few cores to many: A tera-scale computing research overview. http://www.developers.net/intelisdshowcase/view/2181, 2006.</p>

<div id="refer-3"></div>
<p>[3] CyanogenMod Android Rom, 2009. URL http://www.cyanogenmod.com/home/4-1-6-is-here-with-100-more-jet-fuel.</p>

<div id="refer-4"></div>
<p>[4] Apache http server, 2010. URL http://httpd.apache.org.</p>

<div id="refer-5"></div>
<p>[5] Condor high throughput computing system, 2010. URL http://www.cs.wisc.edu/condor/.</p>

<div id="refer-6"></div>
<p>[6] Dovecot, 2010. URL http://www.dovecot.org.</p>

<div id="refer-7"></div>
<p>[7] Sendmail, 2010. URL http://www.sendmail.org.</p>

<div id="refer-8"></div>
<p>[8] Apache tomcat, 2010. URL http://tomcat.apache.org.</p>

<div id="refer-9"></div>
<p>[9] Tornado, 2010. URL http://www.tornadoweb.org.</p>

<div id="refer-10"></div>
<p>[10] Poweredge r910 rack server, 2011. URL http://www.dell.com/us/en/enterprise/servers/poweredge-r910/pd.aspx?refid=poweredge-r910&cs=555&s=biz.</p>

<div id="refer-11"></div>
<p>[11] Yoshihisa Abe, Hiroshi Yamada, and Kenji Kono. Enforcing appropriate process execution for exploiting idle resources from outside operating systems. In Proceedings of the EuroSys Conference (EuroSys ’08), pages 27–40, Glasgow, Scotland UK, March 2008.</p>

<div id="refer-12"></div>
<p>[12] Michael J. Accetta, Robert V. Baron,William J. Bolosky, David B. Golub, Richard F. Rashid, Avadis Tevanian, and Michael Young. Mach: A new kernel foundation for unix development. In Proceedings of the USENIX Summer Technical Conference (USENIX Summer ’86), Atlanta, Georgia, June 1986.</p>

<div id="refer-13"></div>
<p>[13] Amazon. Amazon elastic compute cloud, 2010. URL http://aws.amazon.com/ec2.</p>

<div id="refer-14"></div>
<p>[14] Thomas E. Anderson, Brian N. Bershad, Edward D. Lazowska, and Henry M. Levy. Scheduler activations: effective kernel support for the user-level management of parallelism. In Proceedings of the 13thACMSymposium on Operating Systems Principles (SOSP ’91), pages 95–109, Pacific Grove, California, October 1991.</p>

<div id="refer-15"></div>
<p>[15] Jeremy Andrews. Cfs and sched yield. Kernel Trap, Sep 2007. URL http://kerneltrap.org/Linux/CFS_and_sched_yield.</p>

<div id="refer-16"></div>
<p>[16] AP. North carolina unemployment claims crash website. USA Today, Jan 2009.</p>

<div id="refer-17"></div>
<p>[17] Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. Information and Control in Gray-Box Systems. In Proceedings of the 18th ACM Symposium on Operating Systems Principles (SOSP ’01), pages 43–56, Banff, Canada, October 2001.</p>

<div id="refer-18"></div>
<p>[18] Andrea C. Arpaci-Dusseau, Remzi H. Arpaci-Dusseau, Nathan C. Burnett, Timothy E. Denehy, Thomas J. Engle, Haryadi S. Gunawi, James A. Nugent, and Florentina I. Popovici. Transforming policies into mechanisms with infokernel. In Proceedings of the 19th ACM Symposium on Operating Systems Principles (SOSP ’03), pages 90–105, Bolton Landing, New York, October 2003.</p>

<div id="refer-19"></div>
<p>[19] Remzi H. Arpaci-Dusseau, and Andrea C. Arpaci-Dusseau. Fail-Stutter Fault Tolerance. In The EighthWorkshop on Hot Topics in Operating Systems (HotOS VIII), pages 33–38, Schloss Elmau, Germany, May 2001.</p>

<div id="refer-20"></div>
<p>[20] Remzi H. Arpaci-Dusseau, and Andrea C. Arpaci-Dusseau. Operating Systems: Four Easy Pieces. 2011.</p>

<div id="refer-21"></div>
<p>[21] Krste Asanovic, Ras Bodik, Bryan Catanzaro, Joseph J. Gebis, Parry Husbands, Kurt Keutzer, David A. Patterson, William L. Plishker, John Shalf, Samuel W. Williams, and Katherine A. Yelick. The Landscape of Parallel Computing Research: A View from Berkeley. Technical Report UCB/EECS-2006-183, University of California, Berkeley, Dec 2006.</p>

<div id="refer-22"></div>
<p>[22] Ozalp Babaoglu, Márk Jelasity, Anne-Marie Kermarrec, Alberto Montresor, and Maarten van Steen. Managing clouds: a case for a fresh look at large unreliable dynamic networks. ACM SIGOPS Operating Systems Review, 40(3):9–13, 2006.</p>

<div id="refer-23"></div>
<p>[23] Ganesh Balakrishnan. Intel xeon 5500 memory peformance. www.crc.nd.edu/rich/Nehalem/Nehalem%20Memory%20performance.pdf.</p>

<div id="refer-24"></div>
<p>[24] Scott A. Banachowski, and Scott A. Brandt. The BEST scheduler for integrated processing of best-effort and soft real-time processes. In Proceedings of Multimedia Computing andNetworking 2002 (MMCN ’02), pages 46–60, San Jose, California, January 2002.</p>

<div id="refer-25"></div>
<p>[25] Scott A. Banachowski, and Scott .A. Brandt. Better real-time response for time-share scheduling. In Proceedings of the 17th International Parallel and Distributed Processing Symposium (IPDPS ’03), Nice, France, April 2003.</p>

<div id="refer-26"></div>
<p>[26] Scott A. Banachowski, JoelWu, and Scott A. Brandt. Missed deadline notification in best-effort schedulers. In Proceedings of Multimedia Computing and Networking 2004 (MMCN ’04), pages 123–135, Santa Clara, California, January 2004.</p>

<div id="refer-27"></div>
<p>[27] Gaurav Banga, Peter Druschel, and Jeffrey C. Mogul. Resource containers: A new facility for resource management in server systems. In Proceedings of the 3rd Symposium on Operating Systems Design and Implementation (OSDI ’99), pages 45–58, New Orleans, Louisiana, February 1999.</p>

<div id="refer-28"></div>
<p>[28] Paul Barham, Simon Crosby, Tim Granger,Neil Stratford, Meriel Huggard, and Fergal Toomey. Measurement based resource allocation for multimedia applications. In Proceedings of ACM/SPIE Multimedia Computing and Networking 1998 (MMCN’98), San Jose, California, January 1998.</p>

<div id="refer-29"></div>
<p>[29] Paul Barham, Boris Dragovic, Keir Fraser, Steven Hand, Tim Harris, Alex Ho, Rolf Neugebauer, Ian Pratt, and AndrewWarfield. Xen and the art of virtualization. In Proceedings of the 19th ACM Symposium on Operating Systems Principles (SOSP ’03), pages 164–177, Bolton Landing, New York, October 2003.</p>

<div id="refer-30"></div>
<p>[30] Paul Barham, Austin Donnelly, Rebecca Isaacs, and Richard Mortier. Using magpie for request extraction and workload modelling. In Proceedings of the 6th Symposium on Operating Systems Design and Implementation (OSDI ’04), pages 259–272, San Francisco, California, December 2004.</p>

<div id="refer-31"></div>
<p>[31] Andrew Baumann, Paul Barham, Pierre-Evariste Dagand, Tim Harris, Rebecca Isaacs, Simon Peter, Timothy Roscoe, Adrian Schupbach, and Akhilesh Singhania. The Multikernel: A New OS Architecture for Scalable Multicore Systems. In Proceedings of the 22nd ACM Symposium on Operating Systems Principles (SOSP ’07), Big Sky, Montana, October 2009.</p>

<div id="refer-32"></div>
<p>[32] Jon Bentley, editor. More programming pearls: confessions of a coder. Addison-Wesley, 1 edition, 1988.</p>

<div id="refer-33"></div>
<p>[33] Brian N. Bershad, Craig Chambers, Susan Eggers, Chris Maeda, Dylan McNamee, Przemyslaw Pardyak, Stefan Savage, and Emin Gun Sirer. Spin—an extensible microkernel for application-specific operating system services. In Proceedings of the 15th ACM Symposium on Operating Systems Principles (SOSP ’95), Copper Mountain Resort, Colorado, December 1995.</p>

<div id="refer-34"></div>
<p>[34] Sergey Blagodurov, Sergey Zhuravlev, and Alexandra Fedorova. Contention Aware Scheduling on Multicore Systems. ACM Transactions on Computer Systems, 28(4), December 2010.</p>

<div id="refer-35"></div>
<p>[35] Daniel Bovet, and Marco Cesati. Understanding the Linux Kernel. O’Reilly Media, Inc., 3rd edition, 2005.</p>

<div id="refer-36"></div>
<p>[36] Silas Boyd-Wickizer, Austin T. Clements, Yandong Mao, Aleksey Pesterev, M. Frans Kaashoek, Robert Morris, and Nickolai Zeldovich. An Analysis of Linux Scalability to Many Cores. In Proceedings of the 9th Symposium on Operating Systems Design and Implementation (OSDI ’10), Vancouver, Canada, December 2010.</p>

<div id="refer-37"></div>
<p>[37] Max Bruning. A Comparison of Solaris, Linux, and FreeBSD Schedulers, October 2005. URL http://www.opensolaris.org/os/article/2005-10-14_a_comparison_of_solaris__linux__and_freebsd_kernels.</p>

<div id="refer-38"></div>
<p>[38] Edouard Bugnion, Scott Devine, and Mendel Rosenblum. Disco: running commodity operating systems on scalable multiprocessors. In Proceedings of the 16th ACM Symposium on Operating Systems Principles (SOSP ’97), pages 143–156, Saint-Malo, France, October 1997.</p>

<div id="refer-39"></div>
<p>[39] Giorgio Buttazzo, and Luca Abeni. Adaptive workload management through elastic scheduling. Real-Time Systems, 23(1/2):7–24, 2002.</p>

<div id="refer-40"></div>
<p>[40] John Calandrino, Dan Baumberger, Jessica Young Tong Li, , and Scott Hahn. Linsched: The linux scheduler simulator. In Proceedings of the 21st ISCA International Conference on Parallel and Distributed Computing and Communication Systems (PDCCS ’08), pages 171–176, Sept 2008.</p>

<div id="refer-41"></div>
<p>[41] George M. Candea, and Michael B. Jones. Vassal: loadable scheduler support for multi-policy scheduling. In Proceedings of the 2nd conference on USENIX Windows NT Symposium, 1998.</p>

<div id="refer-42"></div>
<p>[42] Bryan Cantrill, MichaelW. Shapiro, and Adam H. Leventhal. Dynamic Instrumentation of Production Systems. In Proceedings of the USENIX Annual Technical Conference (USENIX ’04), pages 15–28, Boston, Massachusetts, June 2004.</p>

<div id="refer-43"></div>
<p>[43] Bogdan Caprita, Wong Chun Chan, Jason Nieh, Clifford Stein, and Haoqiang Zheng. Group ratio round-robin: O(1) proportional share scheduling for uniprocessor and multiprocessor systems. In Proceedings of the USENIX Annual Technical Conference (USENIX ’05), pages 337–352, Anaheim, California, April 2005.</p>

<div id="refer-44"></div>
<p>[44] Bogdan Caprita, Jason Nieh, and Clifford Stein. Grouped distributed queues: distributed queue, proportional share multiprocessor scheduling. In Proceedings of the 25th ACM Symposium on Principles of Distributed Computing (PODC ’06), pages 72–81, Denver, Colorado, July 2006.</p>

<div id="refer-45"></div>
<p>[45] Michael J. Carey, Sanjay Krishnamurthi, and Miron Livny. Load control for locking: The ’half-and-half’ approach. In Proceedings of the Ninth Symposium on Principles of Database Systems, pages 72–84, Nashville, Tennessee, April 1990.</p>

<div id="refer-46"></div>
<p>[46] Tracy Carver. Magny-cours and direct connect architecture 2.0, March 2010. URL http://developer.amd.com/documentation/articles/pages/magny-cours-direct-connect-architecture-2.0.aspx.</p>

<div id="refer-47"></div>
<p>[47] Anupam Chanda, Alan L. Cox, and Willy Zwaenepoel. Whodunit: transactional profiling for multi-tier applications. In Proceedings of the EuroSys Conference (EuroSys ’07), pages 17–30, Lisbon, Portugal, March 2007.</p>

<div id="refer-48"></div>
<p>[48] Abhishek Chandra, Micah Adler, Pawan Goyal, and Prashant Shenoy. Surplus fair scheduling: a proportional-share cpu scheduling algorithm for symmetric multiprocessors. In Proceedings of the 4th Symposium on Operating Systems Design and Implementation (OSDI ’00), San Diego, California, October 2000.</p>

<div id="refer-49"></div>
<p>[49] Abhishek Chandra, Pawan Goyal, and Prashant Shenoy. Quantifying the benefits of resource multiplexing in on-demand data centers. In Proceedings of the First ACM Workshop on Algorithms and Architectures for Self-Managing Systems (Self-Manage 2003), June 2003.</p>

<div id="refer-50"></div>
<p>[50] Kevin Closson. Intel xeon 5500 (nehalem EP) NUMA versus interleaved memory (aka SUMA): There is no difference! a forced confession, August 2009. URL http://kevinclosson.wordpress.com/2009/08/14/.</p>

<div id="refer-51"></div>
<p>[51] Jonathan Corbet. Ks2009: How google uses linux. LWN.net, Oct 2009. URL http://lwn.net/Articles/357658/.</p>

<div id="refer-52"></div>
<p>[52] Tommaso Cucinotta, Fabio Checconi, Luca Abeni, and Luigi Palopoli. Self-tuning schedulers for legacy real-time applications. In Proceedings of the Eurosys Conference (EuroSys ’10), pages 55–68, Paris, France, April 2010.</p>

<div id="refer-53"></div>
<p>[53] Timothy E. Denehy, John Bent, Florentina I. Popovici, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. Deconstructing Storage Arrays. In Proceedings of the 11th International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS XI), pages 59–71, Boston, Massachusetts, October 2004.</p>

<div id="refer-54"></div>
<p>[54] Peter J. Denning. Thrashing: its causes and prevention. In AFIPS ’68: Proceedings of the Fall joint computer conference, part I, pages 915–922, 1968.</p>

<div id="refer-55"></div>
<p>[55] John R. Douceur, andWilliam J. Bolosky. Progress-based regulation of low-importance processes. In Proceedings of the 17th ACM Symposium on Operating Systems Principles (SOSP ’99), pages 247–260, Kiawah Island Resort, South Carolina, December 1999.</p>

<div id="refer-56"></div>
<p>[56] Kenneth J. Duda, and David R. Cheriton. Borrowed-virtual-time (bvt) scheduling: supporting latency-sensitive threads in a general-purpose 190 scheduler. In Proceedings of the 17th ACM Symposium on Operating Systems Principles (SOSP ’99), pages 261–276, Kiawah Island Resort, South Carolina, December 1999.</p>

<div id="refer-57"></div>
<p>[57] Frank C. Eigler, Vara Prasad,Will Cohen, Hien Nguyen, Martin Hunt, Jim Keniston, and Brad Chen. Architecture of systemtap: a Linux trace/probe tool, July 2005. URL http://sourceware.org/systemtap/archpaper.pdf.</p>

<div id="refer-58"></div>
<p>[58] Jeremy Elson, and Jon Howell. Handling flash crowds from your garage. In Proceedings of the USENIX Annual Technical Conference (USENIX ’08), pages 171–184, Boston, Massachusetts, June 2008.</p>

<div id="refer-59"></div>
<p>[59] Dawson R. Engler, M. Frans Kaashoek, and James OâŁ™Toole Jr. Exokernel: an operating system architecture for application-level resource management. In Proceedings of the 16th ACM Symposium on Operating Systems Principles (SOSP ’97), pages 251–266, Saint-Malo, France, October 1997.</p>

<div id="refer-60"></div>
<p>[60] D. H. J. Epema. An analysis of decay-usage scheduling in multiprocessors. In Proceedings of the 1995 ACM SIGMETRICS Conference on Measurement and Modeling of Computer Systems (SIGMETRICS ’95), Banff, Alberta, Canada, June 1995.</p>

<div id="refer-61"></div>
<p>[61] Alexandra Fedorova, Margo Seltzer, Christopher Small, and Daniel Nussbaum. Performance of Multithreaded Chip Multiprocessors And Implications For Operating System Design. In Proceedings of the USENIX Annual Technical Conference (USENIX ’05), Anaheim, California, April 2005.</p>

<div id="refer-62"></div>
<p>[62] Laurie J. Flynn. Intel halts development of 2 new microprocessors. The New York Times, May 2004.</p>

<div id="refer-63"></div>
<p>[63] Samuel H. Fuller, and Lynette I. Miller, editors. The Future of Computing Performance: Game Over or Next Level? The National Academies Press, 2011.</p>

<div id="refer-64"></div>
<p>[64] Corey Gough, Suresh Siddha, and Ken Chen. Kernel Scalability – Expanding the horizon beyond fine grain locks. In Linux Symposium, volume 1, pages 153–166, 2007.</p>

<div id="refer-65"></div>
<p>[65] Pawan Goyal, Xingang Guo, and Harrick M. Vin. A hierarchial cpu scheduler for multimedia operating systems. In Proceedings of the 2nd Symposium on Operating Systems Design and Implementation (OSDI ’96), pages 107–121, Seattle,Washington, October 1996.</p>

<div id="refer-66"></div>
<p>[66] Steven D. Gribble. Robustness in complex systems. In The EighthWorkshop on Hot Topics in Operating Systems (HotOS VIII), pages 21 – 26, Schloss Elmau, Germany, May 2001.</p>

<div id="refer-67"></div>
<p>[67] Varun Gupta, and Mor Harchol-Balter. Self-adaptive admission control policies for resource-sharing systems. In Proceedings of the 2009 Joint International Conference on Measurement and Modeling of Computer Systems (SIGMETRICS/Performance ’09), pages 311–322, Seattle,Washington, June 2007.</p>

<div id="refer-68"></div>
<p>[68] Andreas Haeberlen. A case for the accountable cloud. In The 3rd ACM SIGOPS InternationalWorkshop on Large Scale Distributed Systems and Middleware (LADIS’09), October 2009.</p>

<div id="refer-69"></div>
<p>[69] Per Brinch Hansen. The nucleus of a multiprogramming system. Communications of the ACM, 13(4):238–241, April 1970.</p>

<div id="refer-70"></div>
<p>[70] Gernot Heiser. Inside L4/MIPS: Anatomy of a High-Performance Microkernel. School of Computer Science and Engineering, University of NSW, Sydney 2052, Australia, Jan 2001.</p>

<div id="refer-71"></div>
<p>[71] Joseph L Hellerstein. Achieving service rate objectives with decay usage scheduling. IEEE Transactions on Software Engineering, 19(8):813–825, 1993.</p>

<div id="refer-72"></div>
<p>[72] Steven Hofmeyr, Costin Iancu, and Filip Blagojevi´c. Load balancing on speed. In 15th ACM SIGPLAN Annual Symposium on Principles and Practice of Parallel Programming (PPoPP ’10), pages 147–158, January 2010.</p>

<div id="refer-73"></div>
<p>[73] Raj Jain. Congestion control and traffic management in atm networks: Recent advances and a survey. Computer Networks and ISDN Systems, 28 (13):1723 – 1738, 1996.</p>

<div id="refer-74"></div>
<p>[74] Nikolai Joukov, Avishay Traeger, Rakesh Iyer, Charles P.Wright, and Erez Zadok. Operating system profiling via latency analysis. In Proceedings of the 7th Symposium on Operating Systems Design and Implementation (OSDI ’06), pages 89–102, Seattle, Washington, November 2006. </p>

<div id="refer-75"></div>
<p>[75] M. Frans Kaashoek, Dawson R. Engler, Gregory R. Ganger, HÃ©ctor M. Briceño, Russell Hunt, David Mazières, Thomas Pinckney, Robert Grimm, John Jannotti, and Kenneth Mackenzie. Application Performance and Flexibility on Exokernel Systems. In Proceedings of the 16th ACM Symposium on Operating Systems Principles (SOSP ’97), pages 52–65, Saint-Malo, France, October 1997.</p>

<div id="refer-76"></div>
<p>[76] J. Kay, and P. Lauder. A fair share scheduler. Communications of the ACM, 31(1):44–55, 1988.</p>

<div id="refer-77"></div>
<p>[77] Vahid Kazempour, Alexandra Fedorova, and Pouya Alagheband. Performance implications of cache affinity on multicore processors. In Euro-Par ’08, pages 151–161, 2008. </p>

<div id="refer-78"></div>
<p>[78] Con Kolivas. BFS – The Brain Fuck Scheduler. http://ck.kolivas.org/patches/bfs/sched-BFS.txt.</p>

<div id="refer-79"></div>
<p>[79] Con Kolivas. Faqs about bfs v0.310, Nov 2009. URL http://ck.kolivas.org/patches/bfs/bfs-faq.txt.</p>

<div id="refer-80"></div>
<p>[80] Con Kolivas. BFS CPU scheduler v0.304 stable release. LWN.net, Oct 2009. URL http://lwn.net/Articles/357451/.</p>

<div id="refer-81"></div>
<p>[81] Charles Krasic, Mayukh Saubhasik, Anirban Sinha, and Ashvin Goel. Fair and timely scheduling via cooperative polling. In Proceedings of the EuroSys Conference (EuroSys ’09), pages 103–116, Nuremburg, Germany, April 2009.</p>

<div id="refer-82"></div>
<p>[82] Orran Krieger, Marc Auslander, Bryan Rosenburg, RobertW.Wisniewski, Jimi Xenidis, Dilma Da Silva, Michal Ostrowski, Jonathan Appavoo, Maria Butrico, Mark Mergen, Amos Waterland, and Volkmar Uhlig. K42: building a complete operating system. In Proceedings of the EuroSys Conference (EuroSys ’06), pages 133–145, Leuven, Belgium, April 2006.</p>

<div id="refer-83"></div>
<p>[83] Avinesh Kumar. Multiprocessing with the completely fair scheduler. IBM developerWorks, Jan 2008.</p>

<div id="refer-84"></div>
<p>[84] Horacio Andrés Lagar-Cavilla, Joseph Andrew Whitney, Adin Matthew Scannell, Philip Patchin, Stephen M. Rumble, Eyal de Lara, Michael Brudno, and Mahadev Satyanarayanan. Snowflock: rapid virtual machine cloning for cloud computing. In Proceedings of the EuroSys Conference (EuroSys ’09), pages 1–12, Nuremburg, Germany, April 2009.</p>

<div id="refer-85"></div>
<p>[85] Butler W. Lampson. Hints for computer system design. In Proceedings of the 9th ACM Symposium on Operating System Principles (SOSP ’83), pages 33–48, Bretton Woods, New Hampshire, October 1983.</p>

<div id="refer-86"></div>
<p>[86] Ian Leslie, Derek McAuley, Richard Black, Timothy Roscoe, Paul Barham, David Evers, Robin Fairbairns, and Eoin Hyden. The design and implementation of an operating system to support distributed multimedia applications. IEEE Journal on Selected Areas in Communications, 14(7), 1996.</p>

<div id="refer-87"></div>
<p>[87] Tong Li, Dan Baumberger, and Scott Hahn. Efficient and scalable multiprocessor fair scheduling using distributed weighted round-robin. In 14th ACM SIGPLAN Symposium on Principles and Practice of Parallel Programming (PPoPP ’09), pages 65 – 74, February 2009.</p>

<div id="refer-88"></div>
<p>[88] Richard McDougall, and Jim Mauro. Solaris Internals: Solaris 10 and Open-Solaris Kernel Architecture. Sun Microsystems Press, 2nd edition, 2007.</p>

<div id="refer-89"></div>
<p>[89] Marshall Kirk McKusick, and George V. Neville-Neil. The Design and Implementation of the FreeBSD Operating System. Addison-Wesley, 2005.</p>

<div id="refer-90"></div>
<p>[90] Joe Meehean, and Miron Livny. A service migration case study: Migrating the Condor schedd. In Proceedings of the 38th Midwest Instruction Computing Symposium (MICS ’05), April 2005.</p>

<div id="refer-91"></div>
<p>[91] Jeffrey C. Mogul. Emergent (mis)behavior vs. complex software systems. In Proceedings of the EuroSys Conference (EuroSys ’06), pages 293–304, Leuven, Belgium, April 2006.</p>

<div id="refer-92"></div>
<p>[92] Ingo Molinar. CFS Scheduler, . URL Linux_2.6.36/Documentation/scheduler/sched-design-CFS.txt.</p>

<div id="refer-93"></div>
<p>[93] Ingo Molinar. Goals, Design and Implementation of the newultra-scalable O(1) scheduler, . URL Linux_2.6.18/Documentation/sched-design.txt.</p>

<div id="refer-94"></div>
<p>[94] Gordon E. Moore. Cramming more components onto integrated circuits. Electronics, 38(8), April 1965.</p>

<div id="refer-95"></div>
<p>[95] Satish Narayanasamy, Cristiano Pereira, Harish Patil, Robert Cohn, and Brad Calder. Automatic logging of operating system effects to guide application-level architecture simulation. In Proceedings of the joint international conference on Measurement and modeling of computer systems, SIGMETRICS ’06/Performance ’06, pages 216–227, 2006.</p>

<div id="refer-96"></div>
<p>[96] Netcraft. Operating system share by groups for sites in all locations, January 2009. URL https://ssl.netcraft.com/ssl-sample-report/CMatch/osdv_all.</p>

<div id="refer-97"></div>
<p>[97] Chandandeep Singh Pabla. Completely fair scheduler. Linux Journal, Aug 2009.</p>

<div id="refer-98"></div>
<p>[98] Pradeep Padala, Kai-Yuan Hou, Kang G. Shin, Xiaoyun Zhu, Mustafa Uysal, Zhikui Wang, Sharad Singhal, and Arif Merchant. Automated control of multiple virtualized resources. In Proceedings of the EuroSys Conference (EuroSys ’09), pages 13–26, Nuremburg, Germany, April 2009.</p>

<div id="refer-99"></div>
<p>[99] Jitendra Padhye, and Sally Floyd. Identifying the TCP Behavior of Web Servers. In Proceedings of SIGCOMM ’01, San Diego, California, August 2001.</p>

<div id="refer-100"></div>
<p>[100] Maija Palmer. Surge of goods for sale sparks ebay crash and compensation claims. FT.com (Financial Times), Nov 2009.</p>

<div id="refer-101"></div>
<p>[101] Abhay K. Parekh, and Robert G. Gallager. Ageneralized processor sharing approach to flow control in integrated services networks: the single-node case. IEEE/ACM Transactions on Networking, 1(3):344–357, 1993.</p>

<div id="refer-102"></div>
<p>[102] PCLinuxOS. PCLinuxOS 2010 Edition is now available for download. URL http://www.pclinuxos.com/?p=579.</p>

<div id="refer-103"></div>
<p>[103] Nick Piggin. Less Affine Wakeups, February 2005. URL http://lwn.net/Articles/124982/.</p>

<div id="refer-104"></div>
<p>[104] Vijayan Prabhakaran, Andrea C. Arpaci-Dusseau, and Remzi H. Arpaci-Dusseau. Analysis and Evolution of Journaling File Systems. In Proceedings of the USENIX Annual Technical Conference (USENIX ’05), pages 105–120, Anaheim, California, April 2005.</p>

<div id="refer-105"></div>
<p>[105] Pratap Ramamurthy, Vyas Sekar, Aditya Akella, Balachander Krishnamurthy, and Anees Shaikh. Remote profiling of resource constraints of web servers using mini-flash crowds. In Proceedings of the USENIX Annual Technical Conference (USENIX ’08), pages 185–198, Boston, Massachusetts, June 2008.</p>

<div id="refer-106"></div>
<p>[106] John Regehr. Inferring Scheduling Behavior with Hourglass. In Proceedings of the USENIX Annual Technical Conference (FREENIX Track), Monterey, California, June 2002.</p>

<div id="refer-107"></div>
<p>[107] Jeff Roberson. Ule: a modern scheduler for freebsd. In Proceedings of the 2nd USENIX Conference on BSD, 2003.</p>

<div id="refer-108"></div>
<p>[108] Mark E. Russinovich, David A. Solomon, and Alex Ionescu. Windows Internals: Covering Windows Server 2008 and Windows Vista. Microsoft Press, 5 edition, 2009.</p>

<div id="refer-109"></div>
<p>[109] Bianca Schroeder, and Mor Harchol-Balter. Web servers under overload: How scheduling can help. ACM Transactions on Internet Technology, 6(1): 20–52, 2006.</p>

<div id="refer-110"></div>
<p>[110] Guo Shipeng, and Ken Willis. Snags, again, for china ticket sale. Reuters, May 2008.</p>

<div id="refer-111"></div>
<p>[111] Amit Singh. Mac OS X Internals: A systems approach. Addison-Wesley, 2006.</p>

<div id="refer-112"></div>
<p>[112] Christopher Small, and Margo Seltzer. Vino: An integrated platform for operating system and database research. Technical Report TR-30-94, Harvard, 1994.</p>

<div id="refer-113"></div>
<p>[113] David A. Solomon. Inside Windows NT. Microsoft Programming Series. Microsoft Press, 2nd edition, May 1998.</p>

<div id="refer-114"></div>
<p>[114] Stephen Soltesz, Herbert Pötzl, Marc E. Fiuczynski, Andy Bavier, and Larry Peterson. Container-based operating system virtualization: a scalable, high-performance alternative to hypervisors. In Proceedings of the EuroSys Conference (EuroSys ’07), pages 275–287, Lisbon, Portugal, March 2007.</p>

<div id="refer-115"></div>
<p>[115] Christopher Stewart, Terence Kelly, and Alex Zhang. Exploiting nonstationarity for performance prediction. In Proceedings of the EuroSys Conference (EuroSys ’07), pages 31–44, Lisbon, Portugal, March 2007.</p>

<div id="refer-116"></div>
<p>[116] Ion Stoica, and Hussein Abdel-Wahab. Earliest eligible virtual deadline first: A flexible and accurate mechanism for proportional share resource allocation. Technical Report TR-95-22, Old Dominion University, 1996. </p>

<div id="refer-117"></div>
<p>[117] TOP500. Top500 operating system family share, November 2010. URL http://top500.org/stats/list/36/osfam.</p>

<div id="refer-118"></div>
<p>[118] Josep Torrellas, Andrew Tucker, and Anoop Gupta. Evaluating the Performance of Cache-Affinity Scheduling in Shared-Memory Multiprocessors. Journal of Parallel and Distributed Computing, 24:139–151, 1995. </p>

<div id="refer-119"></div>
<p>[119] Avishay Traeger, Ivan Deras, and Erez Zadok. Darc: dynamic analysis of root causes of latency distributions. In Proceedings of the 2008 ACM SIGMETRICS Conference on Measurement and Modeling of Computer Systems (SIGMETRICS ’08), pages 277–288, Annapolis, Maryland, June 2008.</p>

<div id="refer-120"></div>
<p>[120] Andrew Tucker, Anoop Gupta, and Shigeru Urushibara. The Impact of Operating System Scheduling Policies and Synchronization Methods on the Performance of Parallel Applications. In Proceedings of the 1991 ACM SIGMETRICS Conference on Measurement and Modeling of Computer Systems (SIGMETRICS ’91), San Diego, California, May 1991.</p>

<div id="refer-121"></div>
<p>[121] Bhuvan Urgaonkar, Prashant Shenoy, and Timothy Roscoe. Resource overbooking and application profiling in a shared internet hosting platform. ACM Transactions on Internet Technology, 9(1):1–45, 2009.</p>

<div id="refer-122"></div>
<p>[122] Raj Vaswani, and John Zahorjan. The Implications of Cache Affinity on Processor Scheduling for Multiprogrammed, Shared Memory Multiprocessors. In Proceedings of the 13th ACM Symposium on Operating Systems Principles (SOSP ’91), Pacific Grove, California, October 1991.</p>

<div id="refer-123"></div>
<p>[123] Thiemo Voigt, and Per Gunningberg. Adaptive resource-based web server admission control. In The Seventh IEEE Symposium on Computers and Communications (ISCC ’02), Giardini Naxos, Italy, July 2002.</p>

<div id="refer-124"></div>
<p>[124] Thiemo Voigt, Renu Tewari, Douglas Freimuth, and Ashish Mehra. Kernel mechanisms for service differentiation in overloaded web servers. In Proceedings of the USENIX Annual Technical Conference (USENIX ’02), pages 189–202, Monterey, California, June 2002.</p>

<div id="refer-125"></div>
<p>[125] Carl A. Waldspurger, and William E. Weihl. Lottery scheduling: flexible proportional-share resource management. In Proceedings of the 1st Symposium on Operating Systems Design and Implementation (OSDI ’94), Monterey, California, November 1994.</p>

<div id="refer-126"></div>
<p>[126] Carl A.Waldspurger, andWilliam E.Weihl. Stride scheduling: Deterministic proportional- share resource management. Technical Report TM-528, Massachusetts Institute of Technology, 1995.</p>

<div id="refer-127"></div>
<p>[127] Matt Welsh, David Culler, and Eric Brewer. SEDA: An Architecture for Well-Conditioned, Scalable Internet Services. In Proceedings of the 18th ACM Symposium on Operating Systems Principles (SOSP ’01), Banff, Canada, October 2001.</p>

<div id="refer-128"></div>
<p>[128] Windows. Windows azure platform, 2010. URL http://www.microsoft.com/windowsazure.</p>

<div id="refer-129"></div>
<p>[129] Bruce L. Worthington, Gregory R. Ganger, Yale N. Patt, and John Wilkes. On-line extraction of scsi disk drive parameters. In Proceedings of the joint international conference on Measurement and modeling of computer systems, SIGMETRICS ’95/PERFORMANCE ’95, pages 146–156, 1995.</p>

<div id="refer-130"></div>
<p>[130] Ting Yang, Tongping Liu, Emery D. Berger, Scott F. Kaplan, and J. Eliot B. Moss. Redline: First class support for interactivity in commodity operating systems. In Proceedings of the 8th Symposium on Operating Systems Design and Implementation (OSDI ’08), pages 73–86, San Diego, California, December 2008.</p>

<div id="refer-131"></div>
<p>[131] Kamen Yotov, Keshav Pingali, and Paul Stodghill. Automatic measurement of memory hierarchy parameters. In Proceedings of the 2005 ACM SIGMETRICS Conference on Measurement and Modeling of Computer Systems (SIGMETRICS ’05), pages 181–192, Banff, Canada, June 2005.</p>

<div id="refer-132"></div>
<p>[132] J. Zahorjan, E.D. Lazowska, and D.L. Eager. The Effect of Scheduling Discipline on Spin Overhead in Shared Memory Parallel Processors. IEEE Transactions on Parallel and Distributed System, 2(2):180–198, April 1991.</p>

<div id="refer-133"></div>
<p>[133] Alan Zeichick. Frequently asked questions: NUMA, SMP, and AMDs direct connect architecture, August 2006. URL http://developer.amd.com/pages/810200618.aspx.</p>

<div id="refer-134"></div>
<p>[134] ZenWalk. ZenWalk 6.4 is Ready. URL http://www.zenwalk.org/modules/news/article.php?storyid=107.</p>