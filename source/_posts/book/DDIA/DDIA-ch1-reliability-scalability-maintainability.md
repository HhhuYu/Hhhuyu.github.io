---
title: '[DDIA][第一章]：可靠性、可拓展性、可维护性'
categories:
  - book
  - DDIA
tags:
  - 分布式
  - 可靠性
  - 可拓展性
  - 可维护性
mathjax: false
date: 2021-10-28 23:44:26
---


# 前言

久仰DDIA大名，刚把`MIT 6.824`做完，之后应该会把DDIA的内容做一个摘录，其中也会夹杂着`6.824`的论文翻译和`lab`记录的整理

原书链接: [Designing Data-Intensive Application](https://vonng.gitbooks.io/ddia-cn/content/ch1.html)

# 第一章

## 数据系统

现今很多应用属于数据密集型 （data-intensive），而非计算密集型（compute-intensive）。主要的问题来自数据量、数据的复杂性以及数据的变更速度。

数据密集型应用通常都需要：
- 存储数据，以便自己或其他应用程序之后能再次找到 （`数据库（database）`）
- 记住开销昂贵操作的结果，加快读取速度（`缓存（cache）`）
- 允许用户按关键字搜索数据，或以各种方式对数据进行过滤（`搜索索引（search indexes）`）
- 向其他进程发送消息，进行异步处理（`流处理（stream processing）`）
- 定期处理累积的大批量数据（`批处理（batch processing）`）

这些都属于数据系统（data-system）。


其中三个在大多数软件种都非常重要的问题：
- 可靠性（Reliability）
  - 系统在困境（adversity）（硬件故障、软件故障、人为错误）中仍可正常工作（正确完成功能，并能达到期望的性能水准）。

- 可扩展性（Scalability）
  有合理的办法应对系统的增长（数据量、流量、复杂性）

- 可维护性（Maintainability）
  - 许多不同的人（工程师、运维）在不同的生命周期，都能高效地在系统上工作（使系统保持现有行为，并适应新的应用场景）。

## 可靠性

人们对可靠软件的典型期望包括：
- 应用程序表现出用户所期望的功能。  
- 允许用户犯错，允许用户以出乎意料的方式使用软件。  
- 在预期的负载和数据量下，性能满足要求。  
- 系统能防止未经授权的访问和滥用。  

简单的说可以认为是“即使出现问题，系统也能继续正确工作”

`故障`（fault）：造成错误的原因  
`容错`（fault-tolerant）：能预料并对应故障的系统特性，它是一定范围内的容忍

`故障`（fault）不同于`失效`（failure）
`故障`通常定义为系统的一部分状态偏离其标准，而`失效`则是系统作为一个整体停止向用户提供服务。`故障`的概率不可能降到零，因此最好设计容错机制以防因`故障`而导致`失效`

现在会通过提高触发故障来提高保障率，因为可以通过故意引发故障来确保容错机制的正常运行。

同时相比组织错误，通常倾向与容忍错误。但是在某些环境下，不容忍错误更好。例：银行系统中的出错不能容忍，多种错误累计将对账号造成不可逆的伤害，这时候应该选择直接停止服务。


### 硬件故障

硬盘的`平均无故障时间`（MTTF mean time to failure）约为10-50年

通常是增加单个硬件的冗余度，比如RAID，双路电源，热插拔CPU等

云平台的设计优先考虑`灵活性`和`弹性` ，因为云平台有大量的机器，一旦一个主机失效，那么一般是移动其他机器上运行。

### 软件错误

系统性错误：

- 接受特定的错误输入，便导致所有应用服务器实例崩溃的BUG。例如2012年6月30日的闰秒，由于Linux内核中的一个错误，许多应用同时挂掉了。
- 失控进程会占用一些共享资源，包括CPU时间、内存、磁盘空间或网络带宽。
- 系统依赖的服务变慢，没有响应，或者开始返回错误的响应。
- 级联故障，一个组件中的小故障触发另一个组件中的故障，进而触发更多的故障。

需要系统不断自检，出现差异时报警

### 人为错误

- 以最小化犯错机会的方式设计系统。例如，精心设计的抽象、API和管理后台
- 将人们最容易犯错的地方与可能导致失效的地方解耦（decouple）。
- 在各个层次进行彻底的测试，从单元测试、全系统集成测试到手动测试。
- 允许从人为错误中简单快速地恢复，以最大限度地减少失效情况带来的影响。
- 配置详细和明确的监控，比如性能指标和错误率。
- 良好的管理实践与充分的培训—


## 可拓展性

​ 可扩展性（Scalability） 是用来描述系统应对负载增长能力的术语。

- 说“X可扩展”或“Y不可扩展”. `X`
- “如果系统以特定方式增长，有什么选项可以应对增长？”`√`
- “如何增加计算资源来处理额外的负载？”等问题。`√`

### 描述负载

负载参数：例如：参数的最佳选择取决于系统架构，它可能是每秒向Web服务器发出的请求、数据库中的读写比率、聊天室中同时活跃的用户数量、缓存命中率或其他东西

`fan-out`: 扇出：从电子工程学中借用的术语，它描述了输入连接到另一个门输出的逻辑门数量。 输出需要提供足够的电流来驱动所有连接的输入。 在事务处理系统中，我们使用它来描述为了服务一个传入请求而需要执行其他服务的请求数量

### 描述性能

`吞吐量（throughput）`即每秒可以处理的记录数量，或者在特定规模数据集上运行作业的总时间

`响应时间（response time）`即客户端发送请求到接收响应之间的时间。

>`延迟（latency）` 和 `响应时间（response time）` 经常用作同义词，但实际上它们并不一样。响应时间是客户所看到的，除了实际处理请求的时间（ `服务时间（service time）` ）之外，还包括网络延迟和排队延迟。延迟是某个请求等待处理的`持续时长`，在此期间它处于 `休眠（latent）` 状态，并等待服务。

百分位点：
如果想知道典型场景下用户需要等待多长时间，那么中位数是一个好的度量标准：一半用户请求的响应时间少于响应时间的中位数，另一半服务时间比中位数长。中位数也被称为第50百分位点，有时缩写为p50。注意中位数是关于单个请求的；如果用户同时发出几个请求（在一个会话过程中，或者由于一个页面中包含了多个资源），则至少一个请求比中位数慢的概率远大于50％。

​ 百分位点通常用于`服务级别目标（SLO, service level objectives）`和`服务级别协议（SLA, service level agreements）`，即定义服务预期性能和可用性的合同。 SLA可能会声明，如果服务响应时间的中位数小于200毫秒，且99.9百分位点低于1秒，则认为服务工作正常（如果响应时间更长，就认为服务不达标）。这些指标为客户设定了期望值，并允许客户在SLA未达标的情况下要求退款。

在多重调用的后端服务里，高百分位数变得特别重要。即使并行调用，最终用户请求仍然需要等待最慢的并行调用完成。只需要一个缓慢的调用就可以使整个最终用户请求变慢。即使只有一小部分后端调用速度较慢，如果最终用户请求需要多个后端调用，则获得较慢调用的机会也会增加，因此较高比例的最终用户请求速度会变慢（效果称为尾部延迟放大）。  



![fig1-5](https://cdn.jsdelivr.net/gh/charstal/images/hexo/DDIA-chap1-reliability-scalability-maintainability-fig1-5.png)
### 应对负载的方法

`纵向扩展（scaling up）`（`垂直扩展（vertical scaling）`): 转向更强大的机器

`横向扩展（scaling out）` （`水平扩展（horizontal scaling）`):将负载分布到多台小机器上之间的对立


`弹性（elastic）` 意味着可以在检测到负载增加时自动增加计算资源，而其他系统则是手动扩展（人工分析容量并决定向系统添加更多的机器）。如果负载极难预测（highly unpredictable），则弹性系统可能很有用，但手动扩展系统更简单，并且意外操作可能会更少

## 可维护性

`可操作性（Operability）`

​ 便于运维团队保持系统平稳运行。

`简单性（Simplicity）`

​ 从系统中消除尽可能多的复杂度（complexity），使新工程师也能轻松理解系统。（注意这和用户接口的简单性不一样。）

`可演化性（evolability）`

​ 使工程师在未来能轻松地对系统进行更改，当需求变化时为新应用场景做适配。也称为`可扩展性（extensibility）`，`可修改性（modifiability）`或`可塑性（plasticity）`。

### 可操作性

- 通过良好的监控，提供对系统内部状态和运行时行为的可见性（visibility）
- 为自动化提供良好支持，将系统与标准化工具相集成
- 避免依赖单台机器（在整个系统继续不间断运行的情况下允许机器停机维护）
- 提供良好的文档和易于理解的操作模型（“如果做X，会发生Y”）
- 提供良好的默认行为，但需要时也允许管理员自由覆盖默认值
- 有条件时进行自我修复，但需要时也允许管理员手动控制系统状态
- 行为可预测，最大限度减少意外


### 简单性：管理复杂度

`复杂度（complexity）` 有各种可能的症状，例如：状态空间激增、模块间紧密耦合、纠结的依赖关系、不一致的命名和术语、解决性能问题的Hack、需要绕开的特例等等

​ 简化系统并不一定意味着减少功能；它也可以意味着消除额外的（accidental）的复杂度。

用于消除额外复杂度的最好工具之一是抽象（abstraction）。

### 可演化性：拥抱变化

敏捷（agile） 工作模式为适应变化提供了一个框架

敏捷社区还开发了对在频繁变化的环境中开发软件很有帮助的技术工具和模式，如 测试驱动开发（TDD, test-driven development） 和 重构（refactoring） 。