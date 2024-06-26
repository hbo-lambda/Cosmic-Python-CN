# 第二部分 事件驱动架构


!!! quote
    我很抱歉很久以前就为这个主题创造了“对象”一词，因为它让很多人关注了较小的想法。

    主要思想是“消息传递”。……构建伟大且可扩展的系统的关键在于设计其模块如何通信，而不是它们的内部属性和行为应该是什么。

    <p align='right'>——Alan Kay</p>

能够编写一个领域模型来管理单个业务流程是件好事，但是当我们需要编写多个模型时会发生什么？在现实世界中，我们的应用程序位于组织内，需要与系统的其他部分交换信息。您可能还记得我们在图1中显示的上下文图。

面对这一要求，许多团队选择通过 HTTP API 集成微服务。但如果不小心，他们最终会制造出最混乱的局面：分布式大泥球。

在第二部分中，我们将展示如何将[第一部分](./c.Part1.md)中的技术扩展到分布式系统。我们将缩小范围，看看如何由许多通过异步消息传递进行交互的小组件组成一个系统。

我们将看到我们的服务层和工作单元模式如何允许我们重新配置我们的应用程序以作为异步消息处理器运行，以及事件驱动系统如何帮助我们将聚合和应用程序彼此分离。

<figure markdown='span'>
    ![](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_0102.png){ align=center }
    <figcaption><font size=2>图1.但是所有这些系统究竟如何相互交流呢？</font></figcaption>
</figure>

我们将研究以下模式和技术：

**领域事件**

触发跨越一致性边界的工作流。

**消息总线**

提供从任何端点调用用例的统一方法。

**控制查询规则**

分离读写可以避免事件驱动架构中的尴尬妥协，并提高性能和可扩展性。

另外，我们将添加一个依赖注入框架。这本身与事件驱动架构无关，但它可以解决很多问题。