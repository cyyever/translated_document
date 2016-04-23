# The Google File System
#### Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung<br/><div style="text-align:center">google*</div>

*：这些作者可以通过以下地址联系：{sanjay,hgobioff,shuntak}@google.com

# 摘要（ABSTRACT）

我们设计并实现了Google文件系统，一个可扩展的分布式文件系统，用于大型的数据密集型的分布式应用程序。它运行于不昂贵的硬件上，但提供了容错性，并且它可以提供了高的整体性能（aggregate performance）给大量的客户端。虽然GFS有很多目标和以前的分布式文件系统一样，我们根据对（当前和预计的）应用程序工作负载（workload）和技术环境的观察来引导GFS的设计，这些观察反映我们的设计不同于和以前一些对文件系统的假设。这导致我们重新审视传统的设计选项并探索完全不同的设计要点。

由此产生的文件系统成功地满足了我们的存储需求。它在Google内部被广泛部署，我们那些需要大数据集的服务或研究和开发工作把作为存储平台来生成和处理数据。其中最大的数据集群提供了数百TB的存储能力，这些存储能力由超过一千台机器上的数千个硬盘提供，并且该集群被数百个客户端并发访问。

在本论文里我们提供了为了支持分布式应用而设计的文件系统接口的扩展，从许多方面讨论了我们设计，同时报告了来自于微型基准测试和真实场景的测量结果。
 
# 分类和主题描述符（Categories and Subject Descriptors）
D [4]: 3—分布式文件系统

# 概括术语（General Terms）
设计，可靠性，性能，测量（Design, reliability, performance, measurement）

# 关键词（General Terms）
容错性，可扩展性，数据存储，集群存储（Fault tolerance, scalability, data storage, clustered storage）

## 1. 介绍（INTRODUCTION）
我们设计并实现了Google文件系统(GFS)来满足Google急剧增长的数据处理需求。GFS有很多目标和以前的分布式文件系统一样，例如性能，可扩展性，可靠性和可用性。但是它的设计是根据我们对（当前和预计的）应用程序工作负载（workload）和技术环境的观察来引导的，这些观察反映我们的设计不同于和以前一些对文件系统的设计假设。我们重新审视了传统的设计选项并探索了完全不同的设计要点。

首先，组件的故障是常态而非意外。GFS文件系统包含数百以至数千的由不昂贵的零件组成的存储机器，并且被这些数量级的客户端机器访问。组件的质量和数量事实上导致了在任意给定的时间点总有一些机器出现功能故障并且有一些机器无法从当前的故障中恢复。我们碰到过应用程序BUG，操作系统BUG, 人类错误和磁盘，内存，连接器（connector）和电力供应的故障所导致的问题。因此，持续地监控，错误检测，容错性和自动恢复必须集成到文件系统里。
 
第二，按照传统的标准，我们所涉及的文件是超大文件。几GB的文件是家常便饭。每个文件通常包含许多的程序对象，例如web文档。当我们例行要处理的是快速增长的许多TB的数据集（包含若干亿的程序对象）时，就算是文件系统提供支持，我们也难以有效管理若干亿只有kb大小的文件。
因此，文件系统的设计假定和参数，例如I/O操作和块大小（block size），必须被重新审视。

第三，大多数文件是通过添加（append）新数据而非覆写已有数据来进行修改的。在文件内的随机写几乎是不存在的。一旦写入完毕，文件只被读取，而且经常只被顺序读取。许多类型的数据都有这些特征。有些可能构成大数据仓库以便数据分析程序进行扫描。有些可能是由运行中的程序持续生成的数据流。有些可能是归档数据。有些可能是一台机器产生的中间结果，并同时或者以后被另一台机器处理。由于该超大文件的访问模式，添加（appending）成为了性能优化和原子性保证所关注的焦点，与此同时在客户端缓存数据块的方案不再具有吸引力。

第四，协设计（co-designing）应用程序和文件系统API可以增加我们的灵活性，使得整体系统受益。例如，我们放宽了GFS的一致性模型，这样大大简化了文件系统，但却没有给应用程序增加繁重的负担。我们还引入了一个原子添加的操作以便多个客户端可以并发地添加同一个文件，而不需要在它们之间做额外的同步。我们将在论文后面更详细地讨论这些东西。

目前我们部署了多个GFS集群，用于不同的目的。其中最大的集群拥有超过1000个存储节点，超过300TB的硬盘存储，并且被不同机器上的数百个客户端持续地，繁忙地访问。

## 1. 设计概览（DESIGN OVERVIEW）
### 2.1 假设（Assumptions）
当根据我们的需求设计一个文件系统时，我们被一些假设所指引，这些假设既给予我们挑战，也给予我们机遇。我们在早先聊到了一些关键的观察到的东西，现在我们将更详细地铺陈我们的假设。

* 系统由许多不昂贵的经常故障的组件构造而成。它必须持续的监控自身，并且探测到例常的组件故障，容忍这些故障，并且从中恢复。
* 系统存储适量的大文件。我们期望存储几百万文件，每个有通常100MB或更大。存储几GB的文件对我们来说是常态，因此必须有效地管理它们。必须支持小文件存储，但我们无须为它们做优化。
* 工作负载主要包含两种读操作：大数据量的流式读取（streaming read）和小数据量的随机读取。在大数据量的流式读取里，单个读操作通常读取几百KB，而且经常读取1MB或更多字节。同一个客户端发起的连续操作经常顺序读取一个文件的一个连续区域。一个小数据量的随机读取通常在某个任意的文件偏移量下读取几KB。关注性能的应用程序通常对它们的小数据量读取操作进行排序并批量执行，以便朝着文件尾部的方向稳步地读取而非往返读取。
* 工作负载还包含许多大数据量的顺序写入，它们把数据添加到文件尾部。通常写入的数据量类似于上述流式读取操作的数据量。一旦写入完毕，文件极少再去修改。系统当然也支持在文件任意位置的小数据量的写入，但是这些操作无须高效。
* 系统必须为多个客户端并发地朝同一个文件添加数据的情况来高效地实现良定义（well-defined）的语义。我们的文件经常用作生产者-消费者对列，或者用于多路归并。几百个生产者（每台机器上跑一个生产者）将并发地朝一个文件添加数据。
• The system must efficiently implement well-defined se-
mantics for multiple clients that concurrently append
to the same file. Our files are often used as producer-
consumer queues or for many-way merging. Hundreds
of producers, running one per machine, will concur-
rently append to a file. Atomicity with minimal syn-
chronization overhead is essential. The file may be
read later, or a consumer may be reading through the
file simultaneously.
• High sustained bandwidth is more important than low
latency. Most of our target applications place a pre-
mium on processing data in bulk at a high rate, while
few have stringent response time requirements for an
individual read or write.
