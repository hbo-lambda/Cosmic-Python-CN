# 第一部分: 构建支持领域建模的架构

!!! quote
    <p>“大多数开发人员从未见过领域模型，只见过数据模型。”</p>
    <p align='right'>——Cyrille Martraire</p>
    <p align='right'>DDD EU 2017</p>

我们与大多数开发人员讨论过架构，他们总觉得事情可以变得更好。他们经常试图挽救一个不知何故出错的系统，并试图将一些结构重新放回泥潭。他们知道他们的业务逻辑不应该到处都是，但他们不知道如何修复它。

我们发现，许多开发人员在被要求设计新系统时，会立即开始构建数据库模式，而将对象模型视为事后考虑。这就是一切开始出错的地方。相反，行为应该放在首位，并推动我们的存储需求。毕竟，我们的客户不关心数据模型。他们关心系统做什么；否则他们只会使用电子表格。

本书的第一部分介绍了如何通过TDD构建丰富的对象模型（在[1.领域模型](./d.Domain%20Modeling.md)中），然后我们将展示如何使该模型与技术问题脱钩。我们将展示了如何构建持久化性无关的代码，以及如何围绕我们的领域创建稳定的api，以便我们能够积极地进行重构。

为了实现这一点，我们提出了四个关键的设计模式：

- [存储库模式](./e.Repository%20Pattern.md)，持久存储概念的抽象
- [服务层模式](./g.Flask%20API%20and%20Service%20Layer.md)明确定义用例的开始和结束位置
- [工作单元模式](./i.Unit%20of%20Work%20Pattern.md)提供原子操作
- [聚合模式](./j.aggregates%20and%20Consistency%20Boundaries.md)用语保证数据的完整性

如果您想要一张图来了解我们要实现目标，请查看[第一部分](./c.Part1.md)末尾的应用程序组件图，但如果您还不理解，请不要担心！我们将在本书的这一部分逐个介绍图中的每个框。

<figure markdown='span'>
    ![apwp_p101](https://read-code.oss-cn-beijing.aliyuncs.com/apwp_p101.png){ align=center }
    <figcaption><font size=2>图1.[第一部分] 末尾的应用程序的组件图</font></figcaption>
</figure>

我们还花了一点时间来讨论[耦合和抽象](./f.Coupling%20and%20Abstractions.md)，并通过一个简单的例子来说明我们如何以及为何选择抽象。

三个附录是对第一部分内容的进一步探索：

- [附录B 项目模版](./t.Appendix%20B.md) 是我们示例代码的基础架构的描述：我们如何构建和运行 Docker 镜像，在哪里管理配置信息，以及如何运行不同类型的测试。
- [附录C csvs](./u.Appendix%20C.md) 是一种“布丁的证明”类型的内容，展示了将我们的整个基础设施（Flask API、ORM 和 Postgres）换成涉及 CLI 和 CSV 的完全不同的 I/O 模型是多么容易。
- 最后，如果你想知道如果使用 Django 而不是 Flask 和 SQLAlchemy 这些模式会是什么样子，那么[附录D Django](./v.Appendix%20D.md)可能会让你感兴趣。